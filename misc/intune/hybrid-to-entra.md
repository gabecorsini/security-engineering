# Migration Guide: Hybrid Entra Join to Entra ID Join

This document outlines migration strategies for transitioning devices from Hybrid Entra Joined states to fully Entra ID joined devices with Intune management. The guide provides both Microsoft-supported and third-party approaches.

## Table of Contents
- [Migration Guide: Hybrid Entra Join to Entra ID Join](#migration-guide-hybrid-entra-join-to-entra-id-join)
  - [Table of Contents](#table-of-contents)
  - [Microsoft-Supported Method Using Windows Autopilot](#microsoft-supported-method-using-windows-autopilot)
    - [Overview](#overview)
    - [Prerequisites](#prerequisites)
    - [Migration Process](#migration-process)
      - [1. Collect Device Information for Autopilot Registration](#1-collect-device-information-for-autopilot-registration)
      - [2. Register Devices in Autopilot](#2-register-devices-in-autopilot)
      - [3. Create an Autopilot Deployment Profile](#3-create-an-autopilot-deployment-profile)
      - [4. Create and Assign Configuration Profiles](#4-create-and-assign-configuration-profiles)
      - [5. Prepare Users and Execute Migration](#5-prepare-users-and-execute-migration)
    - [Common Challenges and Solutions](#common-challenges-and-solutions)
    - [Documentation Links](#documentation-links)
  - [ForensiT User State Migration Method](#forensit-user-state-migration-method)
    - [Overview](#overview-1)
    - [Prerequisites](#prerequisites-1)
    - [Migration Process](#migration-process-1)
      - [1. Preparation and Planning](#1-preparation-and-planning)
      - [2. Pre-Migration Tasks](#2-pre-migration-tasks)
      - [3. Execute ForensiT Migration](#3-execute-forensit-migration)
      - [4. Post-Migration Tasks](#4-post-migration-tasks)
    - [Common Challenges and Solutions](#common-challenges-and-solutions-1)
    - [Documentation Links](#documentation-links-1)
  - [Comparison of Approaches](#comparison-of-approaches)
  - [Decision Framework](#decision-framework)
  - [Conclusion](#conclusion)

## Microsoft-Supported Method Using Windows Autopilot

### Overview
Windows Autopilot provides an official Microsoft solution to transition from Hybrid Entra Join to pure Entra ID join. This approach requires a device reset but preserves the hardware while configuring a clean, fully Entra ID joined environment.

### Prerequisites
- **Microsoft Entra ID Premium** license for each user
- **Microsoft Intune** subscription
- **Windows Autopilot** deployment configured in Intune
- **Global Administrator** or **Intune Administrator** permissions
- **Windows 10 version 1809** or later devices
- **Hardware hash collection** from existing devices
- **Network connectivity** to Microsoft services
- **User account readiness** in Entra ID

### Migration Process

#### 1. Collect Device Information for Autopilot Registration

For existing devices, you need to collect the hardware hash to register them in Autopilot:

**Option A: Using Get-WindowsAutopilotInfo PowerShell script**
```powershell
# Install the script (run as administrator)
Install-Script -Name Get-WindowsAutoPilotInfo -Force

# Export device hardware hash to CSV
Get-WindowsAutoPilotInfo -OutputFile C:\AutopilotHashes.csv
```

**Option B: Using Microsoft Endpoint Configuration Manager (MECM/SCCM)**
If you have MECM/SCCM, you can collect Autopilot information at scale through hardware inventory.

**Option C: Using PowerShell and Intune for Bulk Collection**
For organizations with many devices (50+), you can use PowerShell to collect the hardware hashes at scale:

1. **Create a PowerShell script** for remote execution:
```powershell
# Save as Get-AutopilotInfoRemote.ps1
param(
    [Parameter(Mandatory=$true)]
    [string]$OutputFolder,
    
    [Parameter(Mandatory=$true)]
    [string[]]$ComputerNames,
    
    [Parameter(Mandatory=$false)]
    [System.Management.Automation.PSCredential]$Credential
)

# Ensure output folder exists
if (-not (Test-Path $OutputFolder)) {
    New-Item -ItemType Directory -Path $OutputFolder -Force | Out-Null
}

# Loop through computers and collect info
foreach ($computer in $ComputerNames) {
    Write-Host "Processing $computer..."
    $outputFile = Join-Path $OutputFolder "$computer.csv"
    
    $scriptBlock = {
        # Install script if not already available
        if (-not (Get-Command Get-WindowsAutoPilotInfo -ErrorAction SilentlyContinue)) {
            Install-PackageProvider -Name NuGet -Force -Scope CurrentUser
            Install-Script -Name Get-WindowsAutoPilotInfo -Force -Scope CurrentUser
        }
        
        # Get the hardware hash
        Get-WindowsAutoPilotInfo -OutputFile "$env:TEMP\AutopilotHash.csv"
        
        # Return the content
        Get-Content "$env:TEMP\AutopilotHash.csv"
        
        # Clean up
        Remove-Item "$env:TEMP\AutopilotHash.csv" -Force
    }
    
    $params = @{
        ComputerName = $computer
        ScriptBlock = $scriptBlock
    }
    
    if ($Credential) {
        $params.Add('Credential', $Credential)
    }
    
    try {
        $result = Invoke-Command @params
        if ($result) {
            $result | Out-File -FilePath $outputFile -Force
            Write-Host "Successfully collected hardware hash from $computer"
        }
    }
    catch {
        Write-Warning "Failed to collect info from $computer. Error: $_"
    }
}

# Combine all individual CSV files into a single file
$combinedCsv = Join-Path $OutputFolder "Combined_AutopilotInfo.csv"
$header = $null
$csvFiles = Get-ChildItem -Path $OutputFolder -Filter "*.csv" | Where-Object { $_.Name -ne "Combined_AutopilotInfo.csv" }

foreach ($file in $csvFiles) {
    $content = Get-Content -Path $file.FullName
    if (-not $header) {
        $header = $content | Select-Object -First 1
        $header | Out-File -FilePath $combinedCsv -Force
    }
    $content | Select-Object -Skip 1 | Out-File -FilePath $combinedCsv -Append
}

Write-Host "Combined CSV created at: $combinedCsv"
```

2. **Execute the script** against your device list:
```powershell
# Create a list of computers (from AD, CSV, or manually)
$computers = Get-ADComputer -Filter {OperatingSystem -like "Windows*"} | Select-Object -ExpandProperty Name

# Run the script (with optional credential parameter)
$credential = Get-Credential
.\Get-AutopilotInfoRemote.ps1 -OutputFolder "C:\AutopilotData" -ComputerNames $computers -Credential $credential
```

**Option D: Using Scheduled Task Deployment**
For organizations without MECM but with Group Policy access:

1. **Create a Group Policy Object** to deploy a scheduled task
2. Configure the task to run the Get-WindowsAutoPilotInfo script and save results to a network share
3. Set up a collection script on the network share to combine the CSVs

**Option E: Using Microsoft Intune Proactive Remediation**
If devices are already enrolled in Intune:

1. **Create a detection script** that checks if the hardware hash has been uploaded
2. **Create a remediation script** that collects and uploads the hash
3. **Deploy as a Proactive Remediation package** to all devices

#### 2. Register Devices in Autopilot

1. Sign in to [Microsoft Intune admin center](https://intune.microsoft.com)
2. Navigate to **Devices** > **Windows** > **Windows enrollment** > **Devices** (under Windows Autopilot Deployment Program)
3. Select **Import** and upload the CSV file containing the hardware hashes
4. Assign devices to your Autopilot deployment profile

#### 3. Create an Autopilot Deployment Profile

1. In Intune admin center, navigate to **Devices** > **Windows** > **Windows enrollment** > **Deployment Profiles**
2. Create a new profile with the following settings:
   - **Deployment mode**: User-driven
   - **Join to Azure AD as**: Azure AD joined
   - **Microsoft Entra ID tenant domain name**: Your domain
   - **Device name template**: Define your naming convention (optional)
   - **Enable pre-provisioned deployment**: No (for standard reset scenario)

#### 4. Create and Assign Configuration Profiles

Set up the following in Intune:
- Device configuration profiles
- Compliance policies 
- Application deployments

#### 5. Prepare Users and Execute Migration

1. **Back up user data** using OneDrive Known Folders or a similar solution
2. **Document installed applications** for later reinstallation
3. **Communicate the migration plan** to end users
4. **Reset the device** - use the "Reset this PC" option in Windows Settings
5. **During OOBE**, the device will automatically be configured by Autopilot
6. **User signs in** with their Entra ID credentials
7. **Policies and applications deploy** from Intune

### Common Challenges and Solutions

| Challenge | Solution |
|-----------|----------|
| **User data loss** | Ensure OneDrive Known Folders is configured before reset; consider third-party backup solutions |
| **Application reinstallation** | Use Intune to deploy common applications; document specialized software |
| **Autopilot registration delays** | Import hardware hashes well in advance (24-48 hours); verify imports |
| **Network connectivity issues** | Ensure devices have reliable internet access during enrollment |
| **Driver compatibility** | Test process with representative device models; prepare driver packages |
| **User resistance** | Communicate benefits; provide clear documentation; offer support channels |

### Documentation Links
- [Windows Autopilot deployment overview](https://learn.microsoft.com/en-us/mem/autopilot/windows-autopilot)
- [Register devices in Windows Autopilot](https://learn.microsoft.com/en-us/mem/autopilot/add-devices)
- [User-driven Autopilot deployment](https://learn.microsoft.com/en-us/mem/autopilot/user-driven)
- [Microsoft Intune documentation](https://learn.microsoft.com/en-us/mem/intune/)

## ForensiT User State Migration Method

### Overview
ForensiT User State Migration Tool (commonly referred to as User Profile Wizard) provides a non-Microsoft solution that allows for migration from Hybrid Entra Join to Entra ID Join without full device resets, preserving user profiles and application settings.

### Prerequisites
- **ForensiT User Profile Wizard Professional** licenses
- **Local administrator access** on target devices
- **Entra ID accounts** configured for all users
- **Entra ID Join prerequisites** met on target machines
- **Network connectivity** to both on-premises AD and Microsoft Entra ID
- **Workgroup transition capability** in your environment
- **Testing environment** to validate the process

### Migration Process

#### 1. Preparation and Planning

1. **Document the current state** of devices and user profiles
2. **Purchase and download** ForensiT User Profile Wizard Professional
3. **Create a test plan** with representative device samples
4. **Backup critical data** as a precaution

#### 2. Pre-Migration Tasks

1. **Ensure users have Entra ID accounts** properly configured
2. **Verify licenses** are assigned to users
3. **Configure Intune policies** that will apply post-migration
4. **Create deployment packages** for the ForensiT tool

#### 3. Execute ForensiT Migration

1. **Install the User Profile Wizard** on the target computer
2. **Launch the application** with administrator privileges
3. **Configure the migration settings**:
   - Target: Entra ID account (format: user@domain.com)
   - Migration type: Domain to Microsoft Entra ID
   - Profile Handling: Migrate current profile
4. **Execute the migration** process
5. **The tool will**:
   - Disconnect from on-premises domain
   - Move the computer to a workgroup temporarily
   - Join the device to Entra ID
   - Map the local user profile to the Entra ID user account

#### 4. Post-Migration Tasks

1. **Verify Entra ID join status** in System settings
2. **Enroll the device in Intune** (manual or automatic)
3. **Install the Company Portal app** if not automatically deployed
4. **Verify application functionality** and user data preservation
5. **Remove any legacy AD management agents** if necessary

### Common Challenges and Solutions

| Challenge | Solution |
|-----------|----------|
| **Profile corruption** | Back up profiles before migration; have recovery plan |
| **Application compatibility** | Test core applications before full deployment |
| **Credential issues** | Verify Entra ID account access before migration |
| **Group Policy objects** | Transition GPOs to Intune policies before migration |
| **Local admin rights** | Plan admin access strategy post-migration |
| **VPN and network access** | Test network resource access post-migration |
| **Legacy authentication systems** | Identify systems requiring domain authentication |

### Documentation Links
- [ForensiT User Profile Wizard Professional](https://www.forensit.com/domain-migration.html)
- [User Profile Wizard Guide](https://www.forensit.com/downloads/User%20Profile%20Wizard%20Corporate-Professional%20Guide.pdf)
- [Microsoft Entra ID Join documentation](https://learn.microsoft.com/en-us/azure/active-directory/devices/concept-azure-ad-join)
- [Intune device enrollment](https://learn.microsoft.com/en-us/mem/intune/enrollment/device-enrollment)

## Comparison of Approaches

| Factor | Windows Autopilot | ForensiT User Profile Wizard |
|--------|------------------|----------------------------|
| **Data Preservation** | Limited (requires backup) | High (preserves profiles) |
| **Deployment Complexity** | Moderate | Low to Moderate |
| **Time per Device** | 1-2 hours | 30-60 minutes |
| **Cost** | Included with M365 licensing | Additional licensing required |
| **User Disruption** | High | Low to Moderate |
| **Clean State** | Complete refresh | Carries over settings |
| **Microsoft Support** | Fully supported | Not officially supported |
| **Scalability** | High with proper planning | Good for phased rollouts |
| **Risk Level** | Low | Moderate |

## Decision Framework

Consider the following factors when choosing between approaches:

1. **Number of devices** to migrate
2. **Timeline constraints** and urgency
3. **User productivity impact** tolerance
4. **Budget availability** for third-party tools
5. **IT staff resources** and expertise
6. **Desire for clean OS state** vs. continuity
7. **Regulatory requirements** around device management

## Conclusion

Both migration approaches have their merits. The Microsoft Autopilot method provides a supported, clean-slate approach with higher initial user impact but potentially fewer long-term issues. The ForensiT method offers a more seamless transition with lower user disruption but may carry additional complexity and support considerations.

Regardless of the chosen method, thorough testing, clear communication with users, and proper planning will greatly increase the success rate of your migration project.

---

*Last updated: June 25, 2025*