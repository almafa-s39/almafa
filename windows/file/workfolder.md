# Windows Server Work Folders Configuration

> **Note on Server Core:** If you are configuring Work Folders on a Server Core installation, you can perform the GUI provisioning steps by adding the Server Core machine to the Server Manager console of another domain-joined server that has the Desktop Experience (GUI) installed.

## 1. Install Work Folders

Install the Work Folders feature (Sync Share Service) and the required IIS Web Core components.

Open an elevated PowerShell console and run:

```powershell
# Install the Work Folders feature and management tools
Install-WindowsFeature -Name FS-SyncShareService, Web-Server -IncludeManagementTools

# A restart is required to complete the installation
Restart-Computer
```

**Verification:** *(Run after reboot)*

```powershell
# Verify the feature is installed successfully
Get-WindowsFeature -Name FS-SyncShareService | Select-Object Name, InstallState
```

## 2. Provision the Sync Share

Once the server has rebooted, provision the storage location for user data.

1. Open **Server Manager**.
2. Navigate to **File and Storage Services** > **Work Folders**.
3. Click **Tasks** > **New Sync Share...** to launch the wizard.
4. Follow the prompts (Next-Next-Finish) to select the local path, configure user aliases, and apply default security/device policies.

## 3. SSL Certificate Import & Binding

Work Folders requires HTTPS. You must import an SSL certificate (with an exportable private key and the correct DNS Subject Alternative Names) and manually bind it to port 443 on the server.

The following PowerShell script imports the `.pfx` certificate, extracts its thumbprint, generates the required Application ID GUID, and automatically binds the certificate to the default HTTPS port.

```powershell
# 1. Securely convert your certificate password
$pw = ConvertTo-SecureString "<YOUR_CERT_PASSWORD>" -AsPlainText -Force

# 2. Import the PFX certificate into the Local Machine Personal store
# The output is captured into the $cert variable to reuse its Thumbprint
$cert = Import-PfxCertificate -FilePath "C:\path\to\your_certificate.pfx" -Password $pw -CertStoreLocation "Cert:\LocalMachine\My"

# 3. Generate a new random GUID for the Application ID
$appId = New-Guid

# 4. Bind the certificate to port 443 using netsh
# The thumbprint and GUID are automatically injected from the variables above
netsh http add sslcert ipport=0.0.0.0:443 certhash="$($cert.Thumbprint)" appid="{$($appId.Guid)}" certstorename=MY
```

**Verification:**

```powershell
# Verify the certificate was imported into the correct store
Get-ChildItem -Path "Cert:\LocalMachine\My" | Where-Object { $_.Thumbprint -eq $cert.Thumbprint }

# Verify the SSL binding is successfully attached to port 443
netsh http show sslcert ipport=0.0.0.0:443
```
