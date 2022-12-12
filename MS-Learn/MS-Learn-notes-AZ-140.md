# Azure Virtual Desktop notes (AZ-140)

- [Azure Virtual Desktop notes (AZ-140)](#azure-virtual-desktop-notes-az-140)
	- [To check/do](#to-checkdo)
	- [Authentication](#authentication)
		- [Domain join options](#domain-join-options)
			- [Azure AD-joined](#azure-ad-joined)
				- [Limitations](#limitations)
				- [Assign users to host pools](#assign-users-to-host-pools)
				- [Connection options](#connection-options)
		- [Enabling MFA for Azure AD joined virtual machines](#enabling-mfa-for-azure-ad-joined-virtual-machines)
	- [Configuration](#configuration)
		- [Supported Operating Systems](#supported-operating-systems)
		- [Client operating systems](#client-operating-systems)
		- [Host pools](#host-pools)
			- [Configure host pool assignment type (personal desktop host pools)](#configure-host-pool-assignment-type-personal-desktop-host-pools)
		- [Load balancing](#load-balancing)
		- [Update Management](#update-management)
		- [Limits](#limits)
		- [Default RDP file properties](#default-rdp-file-properties)
		- [Licensing](#licensing)
	- [Compute](#compute)
		- [Virtual Machines](#virtual-machines)
			- [Create virtual machines for the host pool](#create-virtual-machines-for-the-host-pool)
			- [Prepare virtual machines](#prepare-virtual-machines)
			- [Register virtual machines to the AVD host pool](#register-virtual-machines-to-the-avd-host-pool)
		- [Virtual Machine Images](#virtual-machine-images)
			- [Managed images](#managed-images)
			- [Upload master image to a storage account in Azure](#upload-master-image-to-a-storage-account-in-azure)
			- [Modify session host image](#modify-session-host-image)
			- [Azure Compute Gallery](#azure-compute-gallery)
				- [Azure Compute Gallery Sharing methods](#azure-compute-gallery-sharing-methods)
				- [Azure Computer Gallery limits](#azure-computer-gallery-limits)
			- [Scaling with Azure Compute Gallery](#scaling-with-azure-compute-gallery)
			- [Azure Image Builder](#azure-image-builder)
				- [Recommendations for creating Windows images](#recommendations-for-creating-windows-images)
				- [Install Microsoft 365 Apps on a master Virtual Hard Disk image](#install-microsoft-365-apps-on-a-master-virtual-hard-disk-image)
				- [Windows Language pack installs](#windows-language-pack-installs)
	- [Storage](#storage)
		- [FSLogix profile containers](#fslogix-profile-containers)
			- [Configure FSLogix profile containers](#configure-fslogix-profile-containers)
			- [Configure the storage account for FSLogix profiles](#configure-the-storage-account-for-fslogix-profiles)
		- [Azure Files](#azure-files)
			- [Supported authentication scenarios](#supported-authentication-scenarios)
			- [Restrictions](#restrictions)
	- [Networking](#networking)
		- [RDP shortpath](#rdp-shortpath)
		- [QoS](#qos)
			- [Quality of Service implementation checklist](#quality-of-service-implementation-checklist)
	- [Performance](#performance)
		- [Host sizing](#host-sizing)
		- [Network](#network)
		- [Performance tooling](#performance-tooling)
	- [Security](#security)
		- [Conditional Access](#conditional-access)
		- [Plan and implement multifactor authentication (MFA) in Azure Virtual Desktop](#plan-and-implement-multifactor-authentication-mfa-in-azure-virtual-desktop)
			- [Conditional Access](#conditional-access-1)
				- [Requirements for an Conditional Access Policy requiring MFA when connection to AVD](#requirements-for-an-conditional-access-policy-requiring-mfa-when-connection-to-avd)
				- [Steps for creating the Conditional Access Policy](#steps-for-creating-the-conditional-access-policy)
			- [RBAC for Azure Virtual Desktop](#rbac-for-azure-virtual-desktop)
			- [Microsoft Intune support](#microsoft-intune-support)
			- [Screen capture protection](#screen-capture-protection)


## To check/do
- [ ] Configuration limits
- [ ] Azure Files vs NetApp files
- [ ] Authentication and device registration
- [ ] Update strategy
- [ ] MSIX App attach

## Authentication
AD DS: Azure Virtual Desktop VMs must domain-join an AD DS service, and the AD DS must be in sync with Azure AD to associate users between the two services. You can use Azure AD Connect to associate AD DS with Azure AD.

### Domain join options

#### Azure AD-joined

The configurations listed below are supported with Azure AD-joined virtual machines:

- Personal desktops with local user profiles.
- Pooled desktops used as a jump box. In this configuration, users first access the Azure Virtual Desktop VM before connecting to a different PC on the network. Users shouldn't save data on the VM.
- Pooled desktops or apps where users don't need to save data on the VM. For example, for applications that save data online or connect to a remote database.

> Microsoft recommends Azure AD-joined virtual machines for scenarios where users only need access to cloud-based resources or Azure AD-based authentication.

##### Limitations

Access to your on-premises or Active Directory domain-joined resources and should be considered when deciding whether Azure AD-joined virtual machines suits your environment. Microsoft recommends Azure AD-joined virtual machines for scenarios where users only need access to cloud-based resources or Azure AD-based authentication.

- Azure Virtual Desktop (classic) doesn't support Azure AD-joined virtual machines.
- Azure AD-joined virtual machines don't currently support external users.
- Azure AD-joined virtual machines only supports local user profiles at this time.
- Azure AD-joined virtual machines can't access Azure Files file shares for FSLogix or MSIX app attach. You'll need Kerberos authentication to access either of these features.
- The Windows Store client doesn't currently support Azure AD-joined virtual machines.
- Azure Virtual Desktop doesn't currently support single sign-on for Azure AD-joined virtual machines.

##### Assign users to host pools

For Azure AD-joined virtual machines, you'll need to do two things:

- Assign your users the Virtual Machine User Login role so they can sign in to the virtual machines.
- Assign administrators who need local administrative privileges the Virtual Machine Administrator Login role.

##### Connection options

To enable access from Windows devices not joined to Azure AD, add `targetisaadjoined:i:1` as a custom RDP property to the host pool.

To access Azure AD-joined virtual machines using the web, Android, macOS and iOS clients, you must add `targetisaadjoined:i:1` as a custom RDP property to the host pool. These connections are restricted to entering user name and password credentials when signing in to the session host.

### Enabling MFA for Azure AD joined virtual machines

You can enable multifactor authentication for Azure AD-joined virtual machines by setting a Conditional Access policy on the Azure Virtual Desktop app. For connections to succeed, disable the legacy per-user multifactor authentication.

## Configuration

### Supported Operating Systems

Azure Virtual Desktop session hosts: A host pool can run the following operating systems:

	* Windows 7 Enterprise
	* Windows 10 Enterprise
	* Windows 10 Enterprise Multi-session
	* Windows Server 2012 R2 and above
	* Custom Windows system images with pre-loaded apps, group policies, or other customizations

### Client operating systems

Azure Virtual Desktop doesn't support the RemoteApp and Desktop Connections (RADC) client or the Remote Desktop Connection (MSTSC) client.

You can access Azure Virtual Desktop resources on devices with Windows 10, Windows 10 IoT Enterprise, and Windows 7 using the Windows Desktop client. ::windows 11?::
The client doesn't support Window 8 or Windows 8.1.

### Host pools

* Pooled host pool
	* Recommended windows 10 multisession
	* No admin rights / unable to add applications 
	* Load balances users to available session host
	
* Personal host pool
	* Usually admin rights 
	* *Always the same session host exclusive to the user 

To create a host pool, workspace and desktop app group, you could use the following PowerShell code:

```powershell
New-AzWvdHostPool -ResourceGroupName <resourcegroupname> -Name <hostpoolname> -WorkspaceName <workspacename> -HostPoolType <Pooled|Personal> -LoadBalancerType <BreadthFirst|DepthFirst|Persistent> -Location <region> -DesktopAppGroupName <appgroupname>
```

To create a registration token to authorize a session host to joint the host pool

```powershell
New-AzWvdRegistrationInfo -ResourceGroupName <resourcegroupname> -HostPoolName <hostpoolname> -ExpirationTime $((get-date).ToUniversalTime().AddDays(1).ToString('yyyy-MM-ddTHH:mm:ss.fffffffZ'))
```

To add AD users to the default desktop app group for the host pool:

```powershell
New-AzRoleAssignment -SignInName <userupn> -RoleDefinitionName "Desktop Virtualization User" -ResourceName <hostpoolname+"-DAG"> -ResourceGroupName <resourcegroupname> -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
```

Or - better - add AD groups to the default app group for the host pool:

```powershell
New-AzRoleAssignment -ObjectId <usergroupobjectid> -RoleDefinitionName "Desktop Virtualization User" -ResourceName <hostpoolname+"-DAG"> -ResourceGroupName <resourcegroupname> -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
```

Export the registration token to a variable, to use later to register the VM to the AVD host pool

```powershell
$token = Get-AzWvdRegistrationInfo -ResourceGroupName <resourcegroupname> -HostPoolName <hostpoolname>
```

#### Configure host pool assignment type (personal desktop host pools)

For personal desktop host pools, there are two assignment options:

1. Automatic assignment
2. Direct assignment

For direct assignment there are 3 cmdlets, for automatic only the first two, but with the automatic assignment type of course.

```powershell
Update-AzWvdHostPool -ResourceGroupName <resourcegroupname> -Name <hostpoolname> -PersonalDesktopAssignmentType Direct
```

```powershell
New-AzRoleAssignment -SignInName <userupn> -RoleDefinitionName "Desktop Virtualization User" -ResourceName <appgroupname> -ResourceGroupName <resourcegroupname> -ResourceType 'Microsoft.DesktopVirtualization/applicationGroups'
```

```powershell
Update-AzWvdSessionHost -HostPoolName <hostpoolname> -Name <sessionhostname> -ResourceGroupName <resourcegroupname> -AssignedUser <userupn>
```

### Load balancing

The following load-balancing methods are available in Azure Virtual Desktop:
		
* Breadth-first load balancing allows you to evenly distribute user sessions across the session hosts in a host pool.
* Depth-first load balancing allows you to saturate a session host with user sessions in a host pool. Once the first session reaches its session limit threshold, the load balancer directs any new user connections to the next session host in the host pool until it reaches its limit, and so on.

### Update Management

There are several options for updating Azure Virtual Desktop desktops. Deploying an updated image every month guarantees compliance and state.

	 
	* [Microsoft Endpoint Configuration Manager (MECM)](https://learn.microsoft.com/en-us/mem/configmgr/)  updates server and desktop operating systems.
	* [Windows Updates for Business](https://learn.microsoft.com/en-us/windows/deployment/update/waas-manage-updates-wufb)  updates desktop operating systems like Windows 10 multi-session.
	* [Azure Update Management](https://learn.microsoft.com/en-us/azure/automation/update-management/overview)  updates server operating systems.
	* [Azure Log Analytics](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/log-analytics-agent)  checks compliance.
	* Deploy a new (custom) image to session hosts every month for the latest Windows and applications updates. You can use an image from the Azure Marketplace or a  [custom Azure managed image](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/capture-image-resource) .

### Limits

Limits to take into consideration

* Azure virtual machine session host name prefixes can't exceed 11 characters, due to auto-assigning of instance names and the NetBIOS limit of 15 characters per computer account.
* You can currently deploy 399 VMs per Azure Virtual Desktop Azure Resource Manager template deployment without Availability Sets, or 200 virtual machines per Availability Set. You can increase the number of VMs per deployment by switching off Availability Sets in either the Azure Resource Manager template or the Azure portal host pool enrollment.

### Default RDP file properties

| **RDP properties**         | **Desktops**                                                        | **RemoteApps**                  |
|----------------------------|---------------------------------------------------------------------|---------------------------------|
| Multi-monitor mode         | Enabled                                                             | N/A                             |
| Drive redirections enabled | Drives, clipboard, printers, COM ports, USB devices, and smartcards | Drives, clipboard, and printers |
| Remote audio mode          | Play locally.                                                       | Play locally.                   |

### Licensing

There are a few ways to use the Azure Virtual Desktop license:

- You can create a host pool and its session host virtual machines using the [Azure Marketplace offering](https://learn.microsoft.com/en-us/azure/virtual-desktop/create-host-pools-azure-marketplace). Virtual machines created this way automatically have the license applied.
- You can create a host pool and its session host virtual machines using the [GitHub Azure Resource Manager template](https://learn.microsoft.com/en-us/azure/virtual-desktop/virtual-desktop-fall-2019/create-host-pools-arm-template). Virtual machines created this way automatically have the license applied.
- You can apply a license to an existing session host virtual machine. Follow the instructions in [Create a host pool with PowerShell](https://learn.microsoft.com/en-us/azure/virtual-desktop/create-host-pools-powershell) to create a host pool and associated virtual machines.

## Compute

### Virtual Machines

#### Create virtual machines for the host pool

You can create a VM in multiple ways:

- Create a virtual machine from an Azure Gallery image.
- Create a virtual machine from a managed image.
- Create a virtual machine from an unmanaged image.

#### Prepare virtual machines

Do the following to prepare your virtual machines before you can install the Azure Virtual Desktop agents and register the virtual machines to your Azure Virtual Desktop host pool:

- Domain join the virtual machine. This allows incoming Azure Virtual Desktop users to be mapped from their Azure Active Directory account to their Active Directory account and be successfully allowed access to the virtual machine.
- Install the Remote Desktop Session Host (RDSH) role if the virtual machine is running a Windows Server OS. The RDSH role allows the Azure Virtual Desktop agents to install properly.

#### Register virtual machines to the AVD host pool

To register the Azure Virtual Desktop agents, do the following on each virtual machine:

- Connect to the virtual machine with the credentials you provided when creating the virtual machine.
- Download and install the Azure Virtual Desktop Agent.
- Download the [Azure Virtual Desktop Agent](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrmXv).
- Run the installer. When the installer asks you for the registration token, enter the value you got from the Get-AzWvdRegistrationInfo cmdlet.
- Download and install the Azure Virtual Desktop Agent Bootloader.
- Download the [Azure Virtual Desktop Agent Bootloader](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RWrxrH).
- Run the installer.

### Virtual Machine Images

Windows 11/10 Enterprise multi-session is available in the Azure Image Gallery. There are two options for customizing this image.

- The first option is to provision a virtual machine in Azure (See: Create a virtual machine from a managed image).
- The second option is to create the image locally by downloading the image, provisioning a Hyper-V virtual machine, and customizing it to suit your needs.

#### Managed images

Before creating a new virtual machine, create a managed virtual machine image to use as the source image and grant read access on the image to any user who should have access to the image.

One managed image supports up to 20 simultaneous deployments. Attempting to create more than 20 virtual machines concurrently, from the same managed image, may result in provisioning timeouts due to the storage performance limitations of a single VHD. To create more than 20 virtual machines concurrently, use a Shared Image Galleries image configured with 1 replica for every 20 concurrent virtual machine deployments.

#### Upload master image to a storage account in Azure

This only applies when the master image was created locally.

The following instructions apply to a master image was created locally that can be loaded into an Azure storage.

1. Convert the VM image (VHD) to Fixed if you haven't already. If you don't convert the image to Fixed, you can't successfully create the image.
2. Upload the VHD to a blob container in your storage account. You can upload quickly with the Storage Explorer tool.
3. Next, go to the Azure portal in your browser and search for "Images." Your search should lead you to the Create image page.

#### Modify session host image

Some steps to take

- Disable automatic updates
- Specify start layout
- Set up time zone redirection
- Disable Storage Sense

#### Azure Compute Gallery

An Azure Compute Gallery is made up of:

- **Gallery** - Container for multiple images. A gallery is deployed in one region.
- **Image definitions** - a conceptual grouping for images.
- **Image versions** - an image type used for deploying a VM or scale set. Image versions can be replicated to other regions where VMs need to be deployed.

##### Azure Compute Gallery Sharing methods

| **Share with:**                                | **Option**                                                                                                                       |
|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Specific people, groups, or service principals | Role-based access control (RBAC) lets you share resources to specific people, groups, or service principals on a granular level. |
| Subscriptions or tenants (Direct Shared)       | Direct shared gallery (preview) lets you share to everyone in a subscription or tenant.                                          |
| Everyone                                       | Community gallery (preview) lets you share your entire gallery publicly, to all Azure users.                                     |

##### Azure Computer Gallery limits

There are limits, per subscription, for deploying resources using Azure Compute Galleries:

- 100 shared image galleries, per subscription, per region.
- 1,000 image definitions, per subscription, per region.
- 10,000 image versions, per subscription, per region.
- 10 image version replicas, per subscription, per region.
- Any disk attached to the image must be less than or equal to 1TB in size.

#### Scaling with Azure Compute Gallery

With Azure Compute Gallery, you can now deploy up to a 1,000 VM instances in a virtual machine scale set (up from 600 with managed images). Image replicas provide for better deployment performance, reliability, and consistency. You can set a different replica count in each target region, based on the scale needs for the region. Since each replica is a deep copy of your image, replicas help scale your deployments linearly with each extra replica. While we understand no two images or regions are the same, here’s our general guideline on how to use replicas in a region:

- For non-Virtual Machine Scale Set deployments - For every 20 VMs that you create concurrently, we recommend you keep one replica. For example, if you're creating 120 VMs concurrently using the same image in a region, we suggest you keep at least 6 replicas of your image.
- For Virtual Machine Scale Set deployments - For every scale set deployment with up to 600 instances, we recommend you keep at least one replica. For example, if you're creating 5 scale sets concurrently, each with 600 VM instances using the same image in a single region, we suggest you keep at least 5 replicas of your image.

We always recommend you to overprovision the number of replicas due to factors like image size, content, and OS type.

![Image Scaling with Azure Compute Gallery](Images/scaling-c888ba94.png)

#### Azure Image Builder

Azure Image Builder uses a Bicep file or an ARM JSON template file to pass information into the Image Builder service. For latest API versions, see template reference. To see examples of full .json files, see the [Azure Image Builder GitHub](https://github.com/Azure/azvmimagebuilder/tree/main/quickquickstarts).

An example of the Bicep template for Image builder:

```powershell
resource azureImageBuilder 'Microsoft.VirtualMachineImages/imageTemplates@2022-02-14' = {
  name: azureImageBuilderName
  location: '<region>'
  tags:{
    <name>: '<value>'
    <name>: '<value>'
  }
  identity:{}
  properties:{
    buildTimeoutInMinutes: <minutes>
    customize: []
    distribute: []
    source: {}
    stagingResourceGroup: '/subscriptions/<subscriptionID>/resourceGroups/<stagingResourceGroupName>'
    validate: {}
    vmProfile:{
      vmSize: '<vmSize>'
      proxyVmSize: '<vmSize>'
      osDiskSizeGB: <sizeInGB>
      vnetConfig: {
        subnetId: '/subscriptions/<subscriptionID>/resourceGroups/<vnetRgName>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<subnetName>'
      }
      userAssignedIdentities: [
        '/subscriptions/<subscriptionID>/resourceGroups/<identityRgName>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identityName1>'
        '/subscriptions/<subscriptionID>/resourceGroups/<identityRgName>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identityName2>'
        '/subscriptions/<subscriptionID>/resourceGroups/<identityRgName>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identityName3>'
        ...
      ]
    }
  }
}
```

For Image Builder to have permissions to read/write images, and read in scripts from Azure Storage, you must create an Azure user-assigned identity that has permissions to the individual resources.

Some things that are good to know:

- Default build timeout is 240 minutes (4 hrs)
- When using customize:
  - You can use multiple customizers.
  - Customizers execute in the order specified in the template.
  - If one customizer fails, then the whole customization component will fail and report back an error.
  - Test the scripts thoroughly before using them in a template. Debugging the scripts by themselves is easier.
  - Don't put sensitive data in the scripts. Inline commands can be viewed in the image template definition. If you have sensitive information (including passwords, SAS token, authentication tokens, etc.), it should be moved into scripts in Azure Storage, where access requires authentication.
  - The script locations need to be publicly accessible, unless you're using MSI.

Examples of IMAGE builder customization settings can be found here: [Create an Azure Image Builder Bicep or ARM JSON template | MS-Learn](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/image-builder-json?tabs=bicep%2Cazure-powershell)

Azure Image Builder supports three distribution targets:

- managedImage - managed image.
- sharedImage - Azure Compute Gallery.
- VHD - VHD in a storage account.

##### Recommendations for creating Windows images

- **VM size**: For Windows, use Standard_D2_v2 or greater. The default size is Standard_D1_v2, which isn't suitable for Windows.
- **Comment your code**: The VM Image Builder log, customization.log, is verbose. If you comment your scripts by using 'write-host', they'll be sent to the logs, which should make troubleshooting easier.
- **Exit codes**: VM Image Builder expects all scripts to return a 0 exit code. If you use a non-zero exit code, VM Image Builder fails the customization and stops the build. If you have complex scripts, add instrumentation and emit exit codes, which will be shown in the customization.log file.
- Test and retest your code on a standalone VM. Ensure that there are no user prompts, that you're using the correct privileges, and so on.
- Networking: Set-NetAdapterAdvancedProperty is set in the optimization script but fails the VM Image Builder build. Because it disconnects the network, it's commented out.

##### Install Microsoft 365 Apps on a master Virtual Hard Disk image

To activate shared computer activation using GPO: `Computer Configuration\Policies\Administrative Templates\Microsoft Office 2016 (Machine)\Licensing`

Sample configuration xml-file:

```xml
<Configuration>
<Add OfficeClientEdition="64" Channel="MonthlyEnterprise">
<Product ID="O365ProPlusRetail">
<Language ID="en-US" />
<Language ID="MatchOS" />
<ExcludeApp ID="Groove" />
<ExcludeApp ID="Lync" />
<ExcludeApp ID="OneDrive" />
<ExcludeApp ID="Teams" />
</Product>
</Add>
<RemoveMSI/>
<Updates Enabled="FALSE"/>
<Display Level="None" AcceptEULA="TRUE" />
<Logging Level="Standard" Path="%temp%\WVDOfficeInstall" />
<Property Name="FORCEAPPSHUTDOWN" Value="TRUE"/>
<Property Name="SharedComputerLicensing" Value="1"/>
</Configuration>
```

Change Office default behaviour:

```shell
rem Mount the default user registry hive
reg load HKU\TempDefault C:\Users\Default\NTUSER.DAT
rem Must be executed with default registry hive mounted.
reg add HKU\TempDefault\SOFTWARE\Policies\Microsoft\office\16.0\common /v InsiderSlabBehavior /t REG_DWORD /d 2 /f
rem Set Outlooks Cached Exchange Mode behavior
rem Must be executed with default registry hive mounted.
reg add "HKU\TempDefault\software\policies\microsoft\office\16.0\outlook\cached mode" /v enable /t REG_DWORD /d 1 /f
reg add "HKU\TempDefault\software\policies\microsoft\office\16.0\outlook\cached mode" /v syncwindowsetting /t REG_DWORD /d 1 /f
reg add "HKU\TempDefault\software\policies\microsoft\office\16.0\outlook\cached mode" /v CalendarSyncWindowSetting /t REG_DWORD /d 1 /f
reg add "HKU\TempDefault\software\policies\microsoft\office\16.0\outlook\cached mode" /v CalendarSyncWindowSettingMonths  /t REG_DWORD /d 1 /f
rem Unmount the default user registry hive
reg unload HKU\TempDefault

rem Set the Office Update UI behavior.
reg add HKLM\SOFTWARE\Policies\Microsoft\office\16.0\common\officeupdate /v hideupdatenotifications /t REG_DWORD /d 1 /f
reg add HKLM\SOFTWARE\Policies\Microsoft\office\16.0\common\officeupdate /v hideenabledisableupdates /t REG_DWORD /d 1 /f
```

Install OneDrive in per-machine mode

```shell
# If you installed Office with OneDrive by omitting <ExcludeApp ID="OneDrive" /, uninstall any existing OneDrive per-user installations from an elevated command prompt by running the following command:

"[staged location]\OneDriveSetup.exe" /uninstall

# Run this command from an elevated command prompt to set the AllUsersInstall registry value:

REG ADD "HKLM\Software\Microsoft\OneDrive" /v "AllUsersInstall" /t REG_DWORD /d 1 /reg:64

# Run this command to install OneDrive in per-machine mode:

Run "[staged location]\OneDriveSetup.exe" /allusers

# Run this command to configure OneDrive to start at sign in for all users:

REG ADD "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v OneDrive /t REG_SZ /d "C:\Program Files (x86)\Microsoft OneDrive\OneDrive.exe /background" /f

# Enable Silently configure user account by running the following command.

REG ADD "HKLM\SOFTWARE\Policies\Microsoft\OneDrive" /v "SilentAccountConfig" /t REG_DWORD /d 1 /f

# Redirect and move Windows known folders to OneDrive by running the following command.

REG ADD "HKLM\SOFTWARE\Policies\Microsoft\OneDrive" /v "KFMSilentOptIn" /t REG_SZ /d "<your-AzureAdTenantId>" /f
```

##### Windows Language pack installs

After installing the language pack in the image and deploying the VM, you will need a logon script for each user that signs in:

Below is an example of such a script.

```powershell
$LanguageList = Get-WinUserLanguageList
$LanguageList.Add("es-es")
$LanguageList.Add("fr-fr")
$LanguageList.Add("zh-cn")
Set-WinUserLanguageList $LanguageList -force
```

## Storage
Azure files recommended for most customers. Other options are Azure NetApp files and Storage Spaces Direct.

See https://learn.microsoft.com/en-us/training/modules/design-user-identities-profiles/4-recommend-appropriate-storage-solution

### FSLogix profile containers

FSLogix addresses many profile container challenges. Key among them are:

- **Performance:** The FSLogix profile containers are high performance and resolve performance issues that have historically blocked cached exchange mode.
- **OneDrive:** Without FSLogix profile containers, OneDrive for Business is not supported in non-persistent RDSH or VDI environments.
- **Additional folders:** FSLogix provides the ability to extend user profiles to include additional folders.

Microsoft has started replacing existing user profile solutions, like UPD, with FSLogix profile containers.

#### Configure FSLogix profile containers

- Create a new virtual machine that will act as a file share
  - Join VM to AD DS
- Create an AD group to host the Virtual Desktop Users
  - Share a new folder on C-drive and add the group created to with Full Control permissions
  - Make a note of the UNC path of the Shared Folder
- Download and install the FSLogix software
  - Configure the registry to enable the profiles:
    - `Computer\HKEY_LOCAL_MACHINE\software\FSLogix`
    - Create a key named `Profiles` and set the following values:
      - `Enabled` = `1`
      - `VHDLocations` = `<network path to file share>`

#### Configure the storage account for FSLogix profiles

Two choices for storage account type: 

- **General purpose version 2 (GPv2) storage accounts:** GPv2 storage accounts allow you to deploy Azure file shares on standard/hard disk-based (HDD-based) hardware. GPv2 storage accounts can store other storage resources such as blob containers, queues, or tables. File shares can be deployed into the transaction optimized (default), hot, or cool tiers.
  - Quota can be set, but you pay only for what you use.
- **FileStorage storage accounts:** FileStorage storage accounts allow you to deploy Azure file shares on premium/solid-state disk-based (SSD-based) hardware. FileStorage accounts store Azure file shares. Storage resources, such as blob containers or queues, cannot be deployed in a FileStorage account.
  - Quota is overloaded to mean provisioned size. The provisioned size is the amount that you will be billed. So you need to calculate future growth and required iOPS in advance.

### Azure Files

#### Supported authentication scenarios

Azure Files supports identity-based authentication for Windows file shares over SMB through the following three methods. You can only use one method per storage account.

- **On-premises AD DS authentication:** On-premises AD DS-joined or Azure AD DS-joined Windows machines can access Azure file shares with on-premises Active Directory credentials that are synched to Azure AD over SMB. Your client must have line of sight to your AD DS. If you already have AD DS set up on-premises or on a VM in Azure where your devices are domain-joined to your AD, you should use AD DS for Azure file shares authentication.
- **Azure AD DS authentication:** Cloud-based, Azure AD DS-joined Windows VMs can access Azure file shares with Azure AD credentials. In this solution, Azure AD runs a traditional Windows Server AD domain on behalf of the customer, which is a child of the customer’s Azure AD tenant.
- **Azure AD Kerberos for hybrid identities:** Using Azure AD for authenticating hybrid user identities allows Azure AD users to access Azure file shares using Kerberos authentication. This means your end users can access Azure file shares over the internet without requiring a line-of-sight to domain controllers from hybrid Azure AD-joined and Azure AD-joined VMs. Cloud-only identities aren't currently supported.

#### Restrictions

- None of the authentication methods support assigning share-level permissions to computer accounts (machine accounts) using Azure RBAC, because computer accounts can't be synced to an identity in Azure AD. If you want to allow a computer account to access Azure file shares using identity-based authentication, use a default share-level permission or consider using a service logon account instead.
- Neither on-premises AD DS authentication nor Azure AD DS authentication is supported against Azure AD-joined devices or Azure AD-registered devices.
- Identity-based authentication isn't supported with Network File System (NFS) shares.

## Networking

### RDP shortpath

Remote Desktop Protocol (RDP) Shortpath for managed networks is a feature of Azure Virtual Desktop that establishes a direct UDP-based transport between Remote Desktop Client and Session host. RDP uses this transport to deliver Remote Desktop and RemoteApp while offering better reliability and consistent latency.

The Azure Virtual Desktop client needs a direct line of sight to the session host. You can get a direct line of sight by using one of these methods:
	
* Make sure the remote client machines are running Windows 11, Windows 10, or Windows 7 and have the  [Windows Desktop client](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/clients/windowsdesktop)  installed. Currently, non-Windows clients aren't supported.
* Use  [ExpressRoute private peering](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-circuit-peerings) .
* Use a  [Site-to-Site virtual private network (VPN) (IPsec-based)](https://learn.microsoft.com/en-us/azure/vpn-gateway/tutorial-site-to-site-portal) 
* Use a  [Point-to-Site VPN (IPsec-based)](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal) 
* Use a  [public IP address assignment](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-network-public-ip-address) .

To enable RDP Shortpath for managed networks, you need to enable the RDP Shortpath listener on the session host. You can enable RDP Shortpath on any number of session hosts used in your environment. However, there's no requirement to enable RDP Shortpath on all hosts in your host pool.

This can be done like this:

1. Install administrative templates that add rules and settings for Azure Virtual Desktop. Download the  [Azure Virtual Desktop policy templates file](https://aka.ms/avdgpo)  (AVDGPTemplate.cab) and extract the contents of the .cab file and .zip archive.
2. Copy the terminalserver-avd.admx file, then paste it into the %windir%\PolicyDefinitions folder.
3. Copy the en-us\terminalserver-avd.admlfile, then paste it into the%windir%\PolicyDefinitions\en-us folder.
4. To confirm the files copied correctly, open the Group Policy Editor and go to Computer Configuration, select Administrative Templates, select Windows Components, select Remote Desktop Services, select Remote Desktop Session Host, and select Azure Virtual Desktop.
5. You should see one or more Azure Virtual Desktop policies.
6. Open the Enable RDP Shortpath for managed networks policy and set it to Enabled. If you enable this policy setting, you can also configure the port number that the Azure Virtual Desktop session host will use to listen for incoming connections. The default port is 3390.
7. Restart your session host to apply the changes.

To allow inbound network traffic for RDP Shortpath, use the Microsoft Defender Firewall with Advanced Security node in the Group Policy Management MMC snap-in to create firewall rules

### QoS

#### Quality of Service implementation checklist
At a high level, do the steps listed to implement QoS:

1. Make sure your network is ready.
2. Make sure that RDP Shortpath for managed networks is enabled - QoS policies aren't supported for reverse connect transport.
3. Implement insertion of DSCP markers on session hosts.

As you prepare to implement QoS, keep the guidelines listed below in mind:

* The shortest path to session host is best.
* Any obstacles in between, such as proxies or packet inspection devices, aren't recommended.

Insert DSCP markers
You can compare DSCP markings to postage stamps that indicate to postal workers how urgent the delivery is and how best to sort it for speedy delivery. Once you've configured your network to give priority to RDP streams, lost packets and late packets should diminish significantly.

We recommend using DSCP value 46 that maps to Expedited Forwarding (EF) DSCP class.


## Performance

### Host sizing

https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/virtual-machine-recs

### Network

Recommended bandwidth

* Light
	* 1.5 Mbps
* Medium
	* 3 Mbps
* Heavy
	5 Mbps
* Power
	15 Mbps

Display resolution vs bandwidth 

Typical display resolutions at 30 fps
Recommended bandwidth

* About 1024 × 768 px
	* 1.5 Mbps
* About 1280 × 720 px
	3 Mbps
* About 1920 × 1080 px
	* 5 Mbps
* About 3840 × 2160 px (4K)
	* 15 Mbps

### Performance tooling

Use the  [Azure Virtual Desktop Experience Estimator](https://azure.microsoft.com/services/virtual-desktop/assessment/)  to determine the connection round-trip time (RTT) from your current location, through the Azure Virtual Desktop service, to the Azure region where you deploy virtual machines.

## Security

### Conditional Access

![Conditional Access Flow](Images/planning-conditional-access-1-060d60dd.png)

### Plan and implement multifactor authentication (MFA) in Azure Virtual Desktop

#### Conditional Access

##### Requirements for an Conditional Access Policy requiring MFA when connection to AVD

- Assign users a license that includes Azure Active Directory Premium P1 or P2.
- An Azure Active Directory group with your users assigned as group members.
- Enable multifactor authentication for all your users.

##### Steps for creating the Conditional Access Policy

1. Sign in to the Azure portal as a global administrator, security administrator, or Conditional Access administrator.
2. Browse to Azure Active Directory > Security > Conditional Access.
3. Select New policy.
4. Give your policy a name. We recommend that organizations create a meaningful standard for the names of their policies.
5. Under Assignments, select Users and groups.
6. Under Include, select Select users and groups > Users and groups > Choose the group you created.
7. Select Done.
8. Under Cloud apps or actions > Include, select Select apps.
9. Select one of the following apps based on which version of Azure Virtual Desktop you're using. Choose Azure Virtual Desktop (App ID 9cdead84-a844-4324-93f2-b2e6bb768d07)
10. Go to Conditions > Client apps, then select where you want to apply the policy to:

- Select Browser if you want the policy to apply to the web client.
- Select Mobile apps and desktop clients if you want to apply the policy to other clients.
- Select both check boxes if you want to apply the policy to all clients.

11. Once you've selected your app, choose Select, and then select Done.
12. Under Access controls > Grant, select Grant access, Require multifactor authentication, and then Select.
13. Under Access controls > Session, select Sign-in frequency, set the value to the time you want between prompts, and then select Select. For example, setting the value to 1 and the unit to Hours, will require multifactor authentication if a connection is launched an hour after the last one.
14. Confirm your settings and set Enable policy to On.
15. Select Create to enable your policy.

#### RBAC for Azure Virtual Desktop

Below are the Azure Virtual Desktop Roles:

- **Desktop Virtualization Contributor role**: Lets you manage all aspects of the deployment. However, it doesn't grant you access to compute resources. You'll also need the User Access Administrator role to publish app groups to users or user groups.
- **Desktop Virtualization Reader role**: Lets you view everything in the deployment but doesn't let you make any changes.
- **The Host Pool Contributor role**: Allows you to manage all aspects of host pools, including access to resources. You'll need an extra contributor role, Virtual Machine Contributor, to create virtual machines. You will need AppGroup and Workspace contributor roles to create host pool using the portal or you can use Desktop Virtualization Contributor role.
- **Host Pool Reader role**: Allows you to view everything in the host pool, but won't allow you to make any changes.
- **Application Group Contributor role**: Lets you manage all aspects of app groups. If you want to publish app groups to users or user groups, you'll need the User Access Administrator role.
- **Application Group Reader role**: Allows you to view everything in the app group and will not allow you to make any changes.
- **Workspace Contributor role**: Allows you to manage all aspects of workspaces. To get information on applications added to the app groups, you'll also need to be assigned the Application Group Reader role.
- **Workspace Reader role**: Lets you view everything in the workspace, but won't allow you to make any changes.
- **User Session Operator role**: Allows you to send messages, disconnect sessions, and use the "logoff" function to sign sessions out of the session host. However, this role doesn't let you perform session host management like removing session host, changing drain mode, and so on. This role can see assignments but can't modify admins. We recommend you assign this role to specific host pools. If you give this permission at a resource group level, the admin will have read permission on all host pools under a resource group.
- **Session Host Contributor role**: Allows you to view and remove session hosts, and change drain mode. They can't add session hosts using the Azure portal because they don't have write permission for host pool objects. If the registration token is valid (generated and not expired), you can use this role to add session hosts to the host pool outside of Azure portal if the admin has compute permissions through the Virtual Machine Contributor role.

There are three RBAC scopes within AVD:

- Host pools
- App groups
- Workspaces

#### Microsoft Intune support

Intune supports Azure Virtual Desktop VM's that are:

- Running Windows 10 Enterprise, version 1809 or later.
- Hybrid Azure AD-joined
- Set up as personal remote desktops in Azure.
- Enrolled in Intune in one of the following methods:
  - Configure Active Directory group policy to automatically enroll devices that are hybrid Azure AD joined.
  - Configuration Manager co-management.
  - User self-enrollment via Azure AD Join.

> Intune doesn't currently support management of Windows 10 Enterprise multi-session.

#### Screen capture protection

The following clients currently support screen capture protection:

-Windows Desktop client supports screen capture protection for full desktops only.
- macOS client version 10.7.0 or later supports screen capture protection for both RemoteApp and full desktops.

You can configure this using a GPO with Azure Virtual Desktop Administrative template