<!-- 
---
title: "FSRM"
author: "Gergő Téringer"
---
-->
# FSRM

File Server Resource Manager (FSRM) is a role service in Windows Server (fully supported in Windows Server 2022 and Server 2025) that enables system administrators to manage and classify data stored on file servers. It provides tools to enforce storage quotas, block specific file types via file screening, generate comprehensive usage reports, and display customized Access-Denied messages to users who lack permissions.

## 1. Installation and Initial Setup

To use FSRM features via PowerShell or the administrative console, you must install the File Server Resource Manager role service along with its management tools.

```powershell
Install-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name FS-Resource-Manager`: Installs the core FSRM service binaries and drivers on the Windows Server host.
- `-IncludeManagementTools`: Installs the FSRM GUI console (fsrm.msc) and the PowerShell module for remote or script-based management.

## 2. Quota Management

Storage quotas allow administrators to limit the space allocated to a directory or volume. FSRM supports two types of quotas:

- `Hard Quota`: Prevents users from saving files after the space limit is reached.
- `Soft Quota`: Allows users to exceed the limit, but triggers configured notifications and automated tasks.

### 2.1 Configuring Quota Templates and Quotas

Best practice dictates creating a reusable Quota Template first, then applying that template to target folder paths.

```powershell
# Create a new 10GB Hard Quota Template
New-FsrmQuotaTemplate -Name "10GB Hard Quota" -Size 10GB -Description "Enforces a strict 10GB storage limit."

# Apply the Quota Template to a shared directory
New-FsrmQuota -Path "C:\Shares\Department" -Template "10GB Hard Quota"
```

**Command Breakdown & Explanation:**

- `New-FsrmQuotaTemplate`: Defines a reusable configuration specifying capacity, threshold alerts, and hard/soft enforcement settings.
- `-Size 10GB`: Sets the maximum storage threshold to 10 Gigabytes.
- `New-FsrmQuota`: Binds the specified quota template to a concrete path on the file system.

> [!TIP]
> When defining quotas for user home directories, use Auto Apply Quotas (New-FsrmAutoQuota). This automatically creates individual quotas for existing and newly generated subfolders without needing to apply quotas manually to each user's folder.

## 3. File Screening Management

File screening prevents users from storing unauthorized file types (such as executables, audio/video files, or ransomware-associated extensions) on network shares.

- `Active Screening`: Blocks users from saving restricted file types to disk.
- `Passive Screening`: Allows users to save files, but logs the event or sends an administrative warning.

### 3.1 Defining File Groups and Screens

First, define a File Group specifying the file extension patterns to match. Next, create a File Screen applied to the target path.

```powershell
# Create a File Group containing executable and script patterns
New-FsrmFileGroup -Name "Executables and Scripts" -IncludePattern "*.exe", "*.bat", "*.cmd", "*.ps1", "*.vbs"

# Apply an Active File Screen to block these files on a shared directory
New-FsrmFileScreen -Path "C:\Shares\Department" -IncludeGroup "Executables and Scripts" -Active
```

**Command Breakdown & Explanation:**

- `New-FsrmFileGroup`: Defines a named collection of file wildcard patterns (e.g., `*.exe`).
- `New-FsrmFileScreen`: Applies the screening policy to the target folder.
- `-Active`: Enforces strict blocking (returns an "Access Denied" error to users attempting to write matching files). Leaving this parameter off creates a passive screen.

> [!IMPORTANT]
> File screening works seamlessly alongside anti-ransomware strategies in Windows Server 2022 and Server 2025. You can create a file group containing known ransomware extensions (e.g., `*.crypto`, `*.locky`) and trigger an automated administrative script upon detection.

## 4. Access-Denied Assistance and Custom Error Messages

Access-Denied Assistance helps users troubleshoot permission issues when trying to open files or folders they do not have access to. Instead of displaying a generic Windows "Access Denied" prompt, FSRM can display a customized message with instructions on how to request permissions or contact IT support.

### 4.1 Enabling Access-Denied Remediation

Access-Denied Assistance must be enabled globally on the file server before individual or folder-specific custom messages take effect.
Enable it using your mouse with: `FSRM(local) > Configure Options > Access-Denied Assistance > Check-in enable, and edit the default custom message` or enable it by this powershell commandlet:

```powershell
# Enable Access-Denied Assistance globally with a default custom message
Set-FsrmAccessDeniedRemediation -EnableAccessDeniedAssistance $true `
  -Message "Access to this resource is restricted. Please submit an IT ticket or contact support@company.com to request access." `
  -AutoHelp $true
```

**Command Breakdown & Explanation:**

- `Set-FsrmAccessDeniedRemediation`: Configures global options for the Access-Denied Assistance feature on the server.
- `-EnableAccessDeniedAssistance $true`: Turns on the custom error message engine.
- `-Message`: Defines the global default text displayed to end users when permissions are denied.
- `-AutoHelp $true`: Enables automated diagnostic help within the client prompt.

### 4.2 Configuring Folder-Specific Custom Messages

You can override the global default message to provide context-specific instructions for sensitive directories (e.g., HR or Finance shares).

In the FSRM GUI:

1. Open FSRM (`fsrm.msc`).
2. Navigate to Classification Management > Classification Properties.
3. Under Actions, click Set Folder Management Properties.
4. Select Property: Access-Denied Assistance Message.
5. Add the target Path and enter the custom text Value.

Alternatively, you can configure folder-specific remediation messages using PowerShell:

```powershell
# Set a custom Access-Denied message for a specific directory path
Set-FsrmAccessDeniedRemediation -Path "C:\Shares\HR" `
  -Message "Access to HR records is restricted to HR personnel. If you require access, contact HR-Admin@company.com."
```

**Command Breakdown & Explanation:**

- `Set-FsrmAccessDeniedRemediation -Path`: Targets a specific folder path to override the server's global default Access-Denied message.
- `-Message`: Specifies the custom text string displayed only when access is denied within that folder path.

## 5. Troubleshooting and Verification

Verifying FSRM configurations involves querying active quotas, checking file screens, and confirming Access-Denied Assistance settings.

### 5.1 Verifying Active Quotas

**Command:** `Get-FsrmQuota -Path "C:\Shares\Department"`

**What it checks and variables to look for:**

- `Path`: Confirms the target directory monitored by the quota (e.g., `C:\Shares\Department`).
- `Size`: Displays the maximum capacity limit in bytes (e.g., `10737418240` for 10 GB).
- `SoftLimit`: Confirms whether the quota is hard (`False`) or soft (`True`).
- `Disabled`: Confirms whether the quota enforcement is active (`False`) or suspended (`True`).

### 5.2 Verifying Active File Screens

**Command:** `Get-FsrmFileScreen -Path "C:\Shares\Department"`

**What it checks and variables to look for:**

- `Path`: Displays the folder path where screening is enforced.
- `Active`: Shows `True` if active blocking is enabled, or `False` if passive auditing is configured.
- `IncludeGroup`: Lists the file groups currently blocked by this screen (e.g., `Executables and Scripts`).

### 5.3 Verifying Access-Denied Remediation Settings

**Command:** `Get-FsrmAccessDeniedRemediation`

**What it checks and variables to look for:**

- `EnableAccessDeniedAssistance`: Must be set to `True` for custom prompts to appear.
- `Message`: Confirms the global default custom message text configured on the server.
- `AutoHelp`: Displays `True` if client-side diagnostic assistance is enabled.

<!-- Created by: Gergő Téringer, 2026 -->