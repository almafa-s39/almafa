<!-- 
---
title: "RDS"
author: "Gergő Téringer"
---
-->
# RDS

Windows Remote Desktop Services (RDS) Installation and Architecture Guide
Markdown

This document provides comprehensive administrative guidance for deploying Microsoft Remote Desktop Services (RDS) on Windows Server 2025 and modern Windows environments. It covers Server Core role service compatibility, deployment options, installation procedures via PowerShell and Graphical User Interface (GUI) management consoles, and verification steps.

> [!NOTE]
> Remote Desktop Services allows users to access session-based desktops, virtual machine-based desktops (VDI), and remote application programs (RemoteApp) hosted on centralized Windows Server instances.

## 1. RDS Role Compatibility on Windows Server Core

Windows Server Core provides a reduced attack surface and lower resource overhead. However, due to graphics and IIS dependencies, not all RDS role services can run natively on Server Core installations.

> [!IMPORTANT]
> The **Remote Desktop Web Access (RD Web Access)** role service requires full web rendering and UI components. It is **not supported** on Windows Server Core and must be installed on a Windows Server instance with the **Desktop Experience** feature enabled.

- **Remote Desktop Session Host (RDSH)**: **Supported** on Server Core. Hosts session-based desktops and RemoteApps.
- **Remote Desktop Connection Broker (RDCB)**: **Supported** on Server Core. Manages incoming client connections, reconnects, and session load balancing.
- **Remote Desktop Licensing (RD Licensing)**: **Supported** on Server Core. Manages Client Access Licenses (CALs) required for RDS connections.
- **Remote Desktop Gateway (RD Gateway)**: **Supported** on Server Core. Routes HTTPS-encapsulated RDP connections from external networks into internal networks.
- **Remote Desktop Virtualization Host (RDVH)**: **Supported** on Server Core. Integrates with Hyper-V to host virtual machine-based VDI desktops.
- **Remote Desktop Web Access (RD Web Access)**: **NOT Supported** on Server Core. Provides web interface access to RemoteApps and desktop collections.

## 2. RDS Deployment Options and Architecture

When installing RDS, Windows Server offers two primary deployment scenarios through Server Manager or PowerShell:

1. **Scenario-Based Deployment**: Initializes a unified, centralized RDS management deployment topology.
   - **Quick Start**: Installs RD Connection Broker, RD Web Access, and RD Session Host on a single server. Automatically provisions a default collection and sample RemoteApps. (Ideal for lab testing or small offices).
   - **Standard Deployment**: Allows distributing RDS role services across multiple dedicated servers for scalability, high availability, and enterprise isolation.

2. **Role-Based or Feature-Based Installation**: Installs individual RDS role services manually on standalone servers without binding them into a unified deployment topology. Required when setting up standalone RD Licensing servers or deploying roles directly onto Server Core.

> [!TIP]
> Scenario-based deployments support two architecture types: **Session-based desktop deployment** (utilizes RD Session Host servers) and **Virtual machine-based desktop deployment** (utilizes Hyper-V RD Virtualization Host pools).

## 3. Scenario-Based RDS Deployment

Scenario-based deployments establish the central database and management framework across member servers in an Active Directory domain environment.

### 3.1 Provisioning Standard RDS Deployment via PowerShell

```PowerShell
# Create a Standard Session-Based RDS Deployment across dedicated domain servers
New-RDSessionDeployment -ConnectionBroker "cb01.corp.contoso.com" `
                        -WebAccessServer "web01.corp.contoso.com" `
                        -SessionHost "rdsh01.corp.contoso.com", "rdsh02.corp.contoso.com"

# Configure RD Licensing Server and Licensing Mode (Per User or Per Device)
Add-RDServer -Server "lic01.corp.contoso.com" -Role "RDS-LICENSING" -ConnectionBroker "cb01.corp.contoso.com"

Set-RDLicenseConfiguration -LicenseServer "lic01.corp.contoso.com" `
                           -Mode "PerUser" `
                           -ConnectionBroker "cb01.corp.contoso.com"
```

**Command Breakdown & Explanation:**

- `New-RDSessionDeployment`: Initializes a centralized RDS management topology across specified domain member servers.
- `-ConnectionBroker`: Assigns the server acting as the session state coordinator and connection broker.
- `-WebAccessServer`: Assigns the server hosting the IIS-based RD Web Access portal (Must be running Server with Desktop Experience).
- `-SessionHost`: Adds one or more host servers responsible for hosting user desktop sessions and application execution.
- `Add-RDServer`: Registers an additional server role (e.g., RD Licensing) into an existing deployment.
- `Set-RDLicenseConfiguration`: Configures the active licensing mode (`PerUser` or `PerDevice`) and binds the central license server.

### 3.2 Provisioning Standard RDS Deployment via GUI

1. Open **Server Manager** on a central management server running Desktop Experience.
2. Click **Manage** > **Add Roles and Features**.
3. Select **Remote Desktop Services installation** and click **Next**.
4. Choose **Standard deployment** (or **Quick Start** for a single-server lab) and click **Next**.
5. Select **Session-based desktop deployment** and click **Next**.
6. On the **Connection Broker** page, select the designated server (`cb01`) and move it to the right pane. Click **Next**.
7. On the **Web Access** page, select the designated server with Desktop Experience (`web01`) and click **Next**.
8. On the **Session Host** page, select one or more host servers (`rdsh01`, `rdsh02`) and click **Next**.
9. Check **Restart the destination servers automatically if required** and click **Deploy**.

## 4. Installing RDS Role Services on Server Core

Because Server Core lacks a local graphical installation wizard, RDS roles (such as RD Session Host or RD Gateway) must be installed either locally via PowerShell or remotely using Server Manager from a management workstation.

### 4.1 Installing Server Core RDS Roles via PowerShell

```PowerShell
# Install RD Session Host and RD Licensing roles locally on Server Core
Install-WindowsFeature -Name RDS-RD-Server, RDS-Licensing -IncludeManagementTools

# Specify the RD Licensing Server and License Mode via WMI/PowerShell on standalone Core RDSH
$Path = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\RDPCoreTS\Licensing"
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "LicensingMode" -Value 4 # 4 = Per User, 2 = Per Device

# Define Licensing Server IP or Hostname
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\TermService\Parameters\LicenseServers" -Force
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\TermService\Parameters\LicenseServers" -Name "SpecifiedLicenseServers" -PropertyType MultiString -Value "lic01.corp.contoso.com"
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name RDS-RD-Server, RDS-Licensing`: Installs the Remote Desktop Session Host service (`RDS-RD-Server`) and licensing service (`RDS-Licensing`) on the local Server Core system.
- `-IncludeManagementTools`: Installs associated RSAT PowerShell module tools.
- `LicensingMode`: Sets terminal server licensing policy at the registry level (`4` for Per-User, `2` for Per-Device) when no Connection Broker is deployed.
- `SpecifiedLicenseServers`: Configures the fallback licensing server FQDN/IP address for session host CAL tracking.

### 4.2 Managing Server Core RDS Roles via Remote Server Manager GUI

1. Log into a Windows 11 workstation or Windows Server management host with **RSAT** installed.
2. Open **Server Manager**, right-click **All Servers**, and choose **Add Servers**.
3. Locate and select the **Server Core** machine name (e.g., `CORE-RDSH01`) to add it to the management pool.
4. Click **Manage** > **Add Roles and Features**.
5. Select **Role-based or feature-based installation**.
6. Select the remote **Server Core** machine from the server pool.
7. Under **Server Roles**, expand **Remote Desktop Services** and select **Remote Desktop Session Host** (or other supported Core roles).
8. Proceed through the wizard and click **Install**.

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate RDS deployment health, licensing availability, and listener status on Windows 11 or Windows Server 2025 using elevated diagnostic cmdlets.

### 5.1 Verify RDS deployment state and role service placement

**Command:** `Get-RDServer -ConnectionBroker "cb01.corp.contoso.com" | Select-Object Server, Roles`

**What it checks and variables to look for:**

- **Server**: Must list all participating RDS infrastructure servers
- **Roles**: Must correctly reflect assigned roles (`RDS-CONNECTION-BROKER`, `RDS-WEB-ACCESS`, `RDS-SESSION-HOST`)

### 5.2 Verify RD Session Host licensing server configuration and mode

**Command:** `Get-RDLicenseConfiguration -ConnectionBroker "cb01.corp.contoso.com"`

**What it checks and variables to look for:**

- **Mode**: Must be `PerUser` or `PerDevice`
- **LicenseServer**: Must list valid reachable licensing servers (e.g., `lic01.corp.contoso.com`)

### 5.3 Verify active RDP listener port and terminal service health

**Command:** `Get-Service -Name TermService | Select-Object Name, Status, StartType`

**What it checks and variables to look for:**

- **Status**: Must be `Running`
- **StartType**: Must be `Automatic`

### 5.4 Verify TCP port 3389 RDP network listening status on Server Core

**Command:** `Get-NetTCPConnection -LocalPort 3389 | Select-Object LocalAddress, LocalPort, State`

**What it checks and variables to look for:**

- **LocalPort**: Must be `3389`
- **State**: Must be `Listen`

<!-- Created by: Gergő Téringer, 2026 -->