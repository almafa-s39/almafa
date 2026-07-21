<!-- 
---
title: "WAP"
author: "Gergő Téringer"
---
 -->
# WAP

Web Application Proxy (WAP) operates as a reverse proxy and Active Directory Federation Services (AD FS) proxy. It allows you to securely publish internal web applications to external users while enforcing pre-authentication via AD FS.

This guide covers the deployment of the WAP role, including critical workarounds required for modern Windows Server versions (like Server 2022/2025) to successfully establish their initial trust with the AD FS backend.

## 1. Prerequisites

WAP acts as a proxy for AD FS, meaning it inherently requires a fully functional AD FS infrastructure to operate.

Additionally, the WAP server must have a copy of the SSL certificate used by your AD FS farm. You must export this certificate (including the private key) from the AD FS server and import it into the local computer's Personal certificate store on the WAP server before beginning the installation.

> [!IMPORTANT]
> When exporting the AD FS certificate to move it to the WAP server, you must export it as a .pfx file so that the private key is included. If the private key is missing, WAP cannot bind the certificate to the HTTPS listener.

## 2. TLS 1.3 Workaround

Due to a known bug in the certificate authentication implementation during the initial WAP-to-ADFS trust establishment phase on modern Windows Server operating systems, TLS 1.3 must be temporarily disabled on the WAP server. If left enabled, the configuration wizard will frequently fail to establish the proxy trust.

You can apply this fix via the Registry Editor, or run the following PowerShell commands:

```powershell
# Define the registry path for the TLS 1.3 Client protocol
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Client"

# Create the registry key if it does not exist
New-Item -Path $regPath -Force | Out-Null

# Create the DWORD values to disable TLS 1.3
New-ItemProperty -Path $regPath -Name "DisabledByDefault" -Value 1 -PropertyType DWORD -Force
New-ItemProperty -Path $regPath -Name "Enabled" -Value 0 -PropertyType DWORD -Force

# Restart the server to apply the Schannel changes
Restart-Computer -Force
```

**Command Breakdown & Explanation:**

- `New-Item`: Creates the necessary subkeys in the Schannel registry provider.
- `New-ItemProperty`: Sets `Enabled` to 0 and `DisabledByDefault` to 1, instructing Windows to fall back to TLS 1.2 when establishing the secure channel with the AD FS server.

> [!NOTE]
> After the WAP server has successfully run the configuration wizard and established its proxy trust with the AD FS server, you can revert these registry keys to re-enable TLS 1.3 for standard client traffic.

## 3. Role Installation & Configuration

Once the server has rebooted from the TLS 1.3 change and the certificate is in the local store, you can install the Web Application Proxy role. In Server Manager, this is found under the "Remote Access" role.

Alternatively, you can automate the installation and configuration via PowerShell.

```powershell
# Install the Remote Access role and WAP feature
Install-WindowsFeature -Name Web-App-Proxy -IncludeManagementTools

# Configure the WAP trust (Replace variables with your actual data)
Install-WebApplicationProxy -CertificateThumbprint "A1B2C3D4E5F67890ABCDEF1234567890" -FederationServiceName "sso.company.com"
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name Web-App-Proxy`: Installs the required binaries and the Remote Access Management console.
- `Install-WebApplicationProxy`: Initiates the proxy trust wizard.
- `-CertificateThumbprint`: The thumbprint of the SSL certificate you imported earlier. WAP will bind this to its external listener.
- `-FederationServiceName`: The FQDN of your AD FS service (e.g., `sso.company.com`).
- Note: When running this command, PowerShell will prompt you for credentials. You must provide the credentials of an account that has Local Administrator privileges on the AD FS server (or a Domain Admin if AD FS is installed on a Domain Controller).

## 4. Troubleshooting and Verification

Verifying the Web Application Proxy involves checking its operational configuration and ensuring the proxy trust with the AD FS server is healthy and successfully renewing.

### 4.1 Verifying WAP Configuration

Use this command to verify that WAP is correctly configured and successfully connected to the Federation Service.

**Command:** `Get-WebApplicationProxyConfiguration`

**What it checks and variables to look for:**

- `ConfigurationChangesPollingIntervalSec`: Displays how often WAP checks the AD FS server for new published applications (default is usually `30`).
- `ADFSSignUrl`: Must correctly display the login URL of your AD FS server (e.g., `https://sso.company.com/adfs/ls/`).
- `ConnectedServersName`: Should list the internal FQDNs of your backend AD FS servers. If this array is empty, the trust is broken.

### 4.2 Checking AD FS Connectivity Events

WAP logs its proxy trust events in a dedicated operational log. This is the first place to look if the configuration wizard fails or if external logins stop working.

**Command:** `Get-WinEvent -LogName "AD FS/Admin" -MaxEvents 10`

**What it checks and variables to look for:**

- **Id**: Look for Event ID `394` or `245`. These indicate successful proxy trust establishment and configuration retrieval.
- Look for Event ID `422`. This indicates the WAP server could not establish a trust with the AD FS server, usually due to certificate trust chain issues, time synchronization failure, or the TLS 1.3 bug mentioned in section 2.

<!-- Created by: Gergő Téringer, 2026 -->