<!-- 
---
title: "Windows Server Work Folders Configuration"
author: "Gergő Téringer"
---
-->
# Windows Server Work Folders Configuration

This document provides administrative procedures for deploying and configuring Microsoft Work Folders on Windows Server 2025 and modern Windows environments. Work Folders allows information workers to synchronize work files across personal and corporate devices while maintaining administrative control over data encryption and security policies.

> [!NOTE]
> Work Folders relies on the Sync Share Service (`FS-SyncShareService`) and IIS Hostable Web Core (`Web-Server`) to process file synchronization endpoints. Transport-layer security (HTTPS over TCP port 443) is required for client connections.

## 1. Work Folders Role Feature Installation

> [!IMPORTANT]
> Installing the Work Folders role service installs dependent IIS Web Server components. A system reboot is required following installation to finalize role deployment.

```PowerShell
# Install the Work Folders feature, IIS Web Core, and management tools
Install-WindowsFeature -Name FS-SyncShareService, Web-Server -IncludeManagementTools

# Restart the computer to complete feature installation
Restart-Computer -Force
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name FS-SyncShareService, Web-Server`: Installs the core Work Folders sync service along with required Web Server (IIS) core dependencies.
- `-IncludeManagementTools`: Installs RSAT utilities and the `SyncShare` PowerShell module.
- `Restart-Computer -Force`: Triggers an immediate system restart to finalize role installation.

### 1.1 Installing Work Folders via GUI

1. Open **Server Manager** and click **Manage** > **Add Roles and Features**.
2. Select **Role-based or feature-based installation** and click **Next**.
3. Select the local target server from the pool and click **Next**.
4. Expand **File and Storage Services** > **File and iSCSI Services**, then check **Work Folders**.
5. When prompted, click **Add Features** to include required IIS Web Core components.
6. Click **Next** through Features, check **Restart the destination server automatically if required**, and click **Install**.

## 2. SSL Certificate Import and HTTPS Port Binding

Work Folders clients require HTTPS encryption by default. You must import an SSL/TLS certificate (with a Subject Alternative Name matching your server's public/private Work Folders FQDN) and bind it to TCP port 443.

> [!WARNING]
> Unencrypted HTTP communication is disabled on Work Folders clients by default. Operating without a valid SSL binding will cause client registration and synchronization handshakes to fail.

```PowerShell
# Securely convert your certificate password
$pw = ConvertTo-SecureString "<YOUR_CERT_PASSWORD>" -AsPlainText -Force

# Import the PFX certificate into the Local Machine Personal store
$cert = Import-PfxCertificate -FilePath "C:\path\to\your_certificate.pfx" -Password $pw -CertStoreLocation "Cert:\LocalMachine\My"

# Generate a new random GUID for the Application ID
$appId = New-Guid

# Bind the certificate thumbprint to port 443 using netsh
netsh http add sslcert ipport=0.0.0.0:443 certhash="$($cert.Thumbprint)" appid="{$($appId.Guid)}" certstorename=MY
```

**Command Breakdown & Explanation:**

- `ConvertTo-SecureString`: Encrypts plain-text password strings for secure input into certificate cmdlets.
- `Import-PfxCertificate`: Imports the `.pfx` file containing the certificate and private key into the local computer's Personal (`MY`) certificate store.
- `New-Guid`: Generates a unique Application ID identifier required by the HTTP Server API (`netsh http`).
- `netsh http add sslcert`: Binds the imported certificate thumbprint (`certhash`) to all network interfaces (`0.0.0.0`) on HTTPS port `443`.

### 2.1 Importing Certificate via GUI

1. Press `Win + R`, type `certlm.msc`, and press **Enter** to open Local Computer Certificates.
2. Expand **Personal**, right-click **Certificates**, and select **All Tasks** > **Import...**.
3. Select the `.pfx` file, enter the password, check **Mark this key as exportable**, and import into the **Personal** store.
4. Execute `netsh http add sslcert ipport=0.0.0.0:443 certhash=<THUMBPRINT> appid={<GUID>}` in an elevated Command Prompt to bind the certificate to port 443.

## 3. Sync Share Provisioning

A Sync Share defines the local physical folder path where user files are stored, access permissions for Active Directory groups, and security policies enforced on client devices.

> [!TIP]
> Enable device security policies such as client file encryption (EFS) and automatic screen lock enforcement during sync share creation to secure synced data on remote devices.

### 3.1 Provisioning Sync Share via PowerShell

```PowerShell
# Create a new Work Folders Sync Share
New-SyncShare -Name "FinanceSyncShare" `
              -LocalPath "C:\WorkFolders\Finance" `
              -User "DOMAIN\Finance-Users" `
              -RequireEncryption $true `
              -PasswordAutoLock $true
```

**Command Breakdown & Explanation:**

- `New-SyncShare`: Provisions a new Work Folders synchronization endpoint.
- `-Name "FinanceSyncShare"`: Sets the administrative name for the sync share.
- `-LocalPath "C:\WorkFolders\Finance"`: Specifies the host server directory storing user data.
- `-User "DOMAIN\Finance-Users"`: Authorizes designated Active Directory security group members to utilize the share.
- `-RequireEncryption $true`: Mandates client-side enterprise file encryption (EFS) on synchronized files.
- `-PasswordAutoLock $true`: Enforces client device lock policies requiring password entry on client devices.

### 3.2 Provisioning Sync Share via GUI

1. Open **Server Manager**, navigate to **File and Storage Services** > **Work Folders**.
2. Click **Tasks** in the top-right corner and select **New Sync Share...**.
3. On the **Select the server and path** page, choose the volume and local path (e.g., `C:\WorkFolders\Finance`) and click **Next**.
4. Select user folder naming preference (`User alias` or `User alias@domain`) and click **Next**.
5. Enter the **Sync Share Name** and description, then click **Next**.
6. On the **Grant sync access** page, click **Add...** to specify allowed Active Directory groups (e.g., `DOMAIN\Finance-Users`) and click **Next**.
7. Select required device security policies (**Encrypt Work Folders** and **Automatically lock screen and require a password**) and click **Next**.
8. Review settings and click **Create**, then click **Close**.

## 4. Verification and Troubleshooting

> [!NOTE]
> Execute these verification cmdlets in an elevated PowerShell terminal on Windows 11 or Windows Server 2025 to validate role installation, certificate bindings, and sync share health.

### 4.1 Verify Work Folders feature installation state

**Command:** `Get-WindowsFeature -Name FS-SyncShareService | Select-Object Name, InstallState`

**What it checks and variables to look for:**

- **Name**: Must be `FS-SyncShareService`
- **InstallState**: Must be `Installed`

### 4.2 Verify SSL certificate import and store placement

**Command:** `Get-ChildItem -Path "Cert:\LocalMachine\My" | Select-Object Subject, Thumbprint, NotAfter`

**What it checks and variables to look for:**

- **Subject**: Must match the Work Folders FQDN (e.g., `CN=workfolders.contoso.com`)
- **Thumbprint**: Must match the target `.pfx` certificate thumbprint
- **NotAfter**: Must display a valid future expiration date

### 4.3 Verify HTTPS port 443 SSL binding

**Command:** `netsh http show sslcert ipport=0.0.0.0:443`

**What it checks and variables to look for:**

- **IP:port**: Must be `0.0.0.0:443`
- **Certificate Hash**: Must match the imported certificate thumbprint
- **Application ID**: Must show the generated GUID string

### 4.4 Verify Sync Share status and security policies

**Command:** `Get-SyncShare -Name "FinanceSyncShare" | Select-Object Name, LocalPath, User, RequireEncryption, State`

**What it checks and variables to look for:**

- **Name**: Must be `FinanceSyncShare`
- **LocalPath**: Must be `C:\WorkFolders\Finance`
- **RequireEncryption**: Must be `True`
- **State**: Must be `Online` or `Normal`

<!-- Created by: Gergő Téringer, 2026 -->