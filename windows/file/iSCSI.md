<!-- 
---
title: "Windows iSCSI"
author: "Gergő Téringer"
---
 -->
# Windows iSCSI

This document provides administrative procedures for provisioning iSCSI Target storage, establishing iSCSI Initiator connections, configuring Windows Firewall policies, and ensuring persistent connections across system reboots on Windows Server 2025 and Windows 11.

> [!NOTE]
> iSCSI (Internet Small Computer System Interface) allows block-level storage sharing over standard IP networks. The server sharing the storage is the **iSCSI Target**, and the client attaching to the storage is the **iSCSI Initiator**.

## 1. iSCSI Target and Virtual Disk Provisioning

To share block storage, an iSCSI Virtual Disk (VHDX) must be created on the target server and mapped to an iSCSI Target configured with client access permissions (using IQN or IP address identifiers).

> [!IMPORTANT]
> The **iSCSI Target Server** role feature must be installed on Windows Server before executing target management commands (`Install-WindowsFeature FS-iSCSITarget-Server`).

### 1.1 Creating an iSCSI Target and Virtual Disk via PowerShell

```PowerShell
# Create an iSCSI virtual disk (VHDX)
New-IscsiVirtualDisk -Path "C:\iSCSIDisks\DataDisk01.vhdx" -SizeBytes 100GB

# Create an iSCSI Target and grant access to the Initiator IQN
New-IscsiServerTarget -TargetName "Target-Data01" `
                      -InitiatorIds "IQN:iqn.1991-05.com.microsoft:client01.corp.contoso.com"

# Map the virtual disk to the iSCSI Target
Add-IscsiVirtualDiskTargetMapping -TargetName "Target-Data01" `
                                  -Path "C:\iSCSIDisks\DataDisk01.vhdx"
```

**Command Breakdown & Explanation:**

- `New-IscsiVirtualDisk`: Creates a virtual hard disk file used as block storage for iSCSI clients.
- `-Path "C:\iSCSIDisks\DataDisk01.vhdx"`: Defines the physical path where the VHDX file resides on the server.
- `-SizeBytes 100GB`: Sets the capacity allocated to the virtual disk.
- `New-IscsiServerTarget`: Provisions a new target endpoint that clients connect to.
- `-TargetName "Target-Data01"`: Unique identifier name for the iSCSI target.
- `-InitiatorIds`: Restricts access exclusively to specified client IQNs (iSCSI Qualified Names) or IP addresses.
- `Add-IscsiVirtualDiskTargetMapping`: Links the VHDX file to the created iSCSI target.

### 1.2 Creating an iSCSI Target and Virtual Disk via GUI

1. Open **Server Manager**, navigate to **File and Storage Services** > **iSCSI**.
2. Click **Tasks** > **New iSCSI Virtual Disk**.
3. Select the storage location/volume and click **Next**.
4. Specify a **Name** (e.g., `DataDisk01`) and disk size, then click **Next**.
5. Select **New iSCSI target** on the Target page and click **Next**.
6. Specify a **Target Name** (e.g., `Target-Data01`) and click **Next**.
7. Click **Add** under Access Servers and specify the client Initiator IQN or IP address.
8. Complete the wizard by reviewing settings and clicking **Create**.

## 2. Windows Firewall Configuration for iSCSI

iSCSI traffic uses TCP port 3260. Appropriate inbound firewall rules must be enabled on the target server, and outbound rules must be permitted on the initiator client.

> [!WARNING]
> Blocking TCP port 3260 prevents clients from discovering targets or establishing session handshakes, resulting in connection timeout errors.

### 2.1 Enabling iSCSI Firewall Rules via PowerShell

```PowerShell
# Enable Inbound iSCSI Target Server firewall group on Target Server
Enable-NetFirewallRule -DisplayGroup "iSCSI Target Server"

# Enable Outbound/Inbound iSCSI Initiator firewall rules on Client
Enable-NetFirewallRule -DisplayGroup "iSCSI Service"
```

**Command Breakdown & Explanation:**

- `Enable-NetFirewallRule`: Activates pre-defined built-in Windows Defender Firewall rule sets.
- `-DisplayGroup "iSCSI Target Server"`: Enables inbound rules for TCP port 3260 on the server hosting storage.
- `-DisplayGroup "iSCSI Service"`: Enables network communication rules for the Microsoft iSCSI Initiator service on client devices.

### 2.2 Enabling iSCSI Firewall Rules via GUI

1. Open **Windows Defender Firewall with Advanced Security** (`wf.msc`).
2. Click **Inbound Rules** in the left console tree.
3. Locate rules grouped under **iSCSI Target Server** or **iSCSI Service**.
4. Right-click the rules and select **Enable Rule**.

## 3. iSCSI Initiator Service Configuration and Persistent Auto-Connect

By default, the Microsoft iSCSI Initiator service (`MSiSCSI`) startup type is set to **Manual**. This causes iSCSI targets to become disconnected following a system restart because the service does not launch automatically at boot time.

> [!CAUTION]
> If the `MSiSCSI` service is not configured for automatic startup, applications relying on network-attached iSCSI volumes (such as SQL Server or Hyper-V) will fail to initialize after host reboots.

### 3.1 Configuring Auto-Start and Persistent Connection via PowerShell

```PowerShell
# Configure the iSCSI Initiator service to start automatically at system boot
Set-Service -Name "MSiSCSI" -StartupType Automatic
Start-Service -Name "MSiSCSI"

# Register the Target Portal (Discovery)
New-IscsiTargetPortal -TargetPortalAddress "10.10.1.50"

# Connect to the target and configure session persistence
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:target-data01-target" `
                   -IsPersistent $true
```

**Command Breakdown & Explanation:**

- `Set-Service -Name "MSiSCSI" -StartupType Automatic`: Ensures the Microsoft iSCSI Initiator service starts during system boot before user login.
- `Start-Service -Name "MSiSCSI"`: Launches the service immediately without requiring a system restart.
- `New-IscsiTargetPortal -TargetPortalAddress "10.10.1.50"`: Queries the iSCSI Target Server IP address to discover available targets.
- `Connect-IscsiTarget`: Establishes an active iSCSI block session.
- `-NodeAddress`: Specifies the target IQN string discovered from the portal.
- `-IsPersistent $true`: Saves the session mapping into the persistent binding list so Windows automatically reconnects the disk upon system restart.

### 3.2 Configuring Auto-Start and Persistent Connection via GUI

**Configuring Service Auto-Start:**

1. Open **Services** (`services.msc`).
2. Locate **Microsoft iSCSI Initiator Service**.
3. Right-click and select **Properties**.
4. Change **Startup type** to **Automatic**.
5. Click **Start** if the service is stopped, then click **OK**.

**Connecting to Target Persistently:**

1. Open **iSCSI Initiator** (`iscsicpl.exe`).
2. In the **Discovery** tab, click **Discover Portal...** and enter the Target Server IP (`10.10.1.50`).
3. Switch to the **Targets** tab, select the discovered target name.
4. Click **Connect**.
5. Ensure **Add this connection to the list of Favorite Targets** is checked (this enforces `-IsPersistent $true`).
6. Click **OK**.

## 4. Verification and Troubleshooting

> [!NOTE]
> Run these diagnostic commands in an elevated PowerShell shell on Windows 11 or Windows Server 2025 to verify service startup configuration, active sessions, and firewall rules.

## 4.1 Verify iSCSI Initiator service startup type and running status

**Command:** `Get-Service -Name MSiSCSI | Select-Object Name, Status, StartType`

**What it checks and variables to look for:**

- **Status**: Must be `Running`
- **StartType**: Must be `Automatic`

### 4.2 Verify active iSCSI session and persistent target binding

**Command:** `Get-IscsiSession | Select-Object SessionIdentifier, TargetNodeAddress, IsConnected, IsPersistent`

**What it checks and variables to look for:**

- **TargetNodeAddress**: Must match target IQN (e.g., `iqn.1991-05.com.microsoft:target-data01-target`)
- **IsConnected**: Must be `True`
- **IsPersistent**: Must be `True`

### 4.3 Verify firewall rules for iSCSI traffic

**Command:** `Get-NetFirewallRule -DisplayGroup "iSCSI Target Server" | Select-Object DisplayName, Enabled, Direction, Action`

**What it checks and variables to look for:**

- **Enabled**: Must be `True`
- **Direction**: Must be `Inbound`
- **Action**: Must be `Allow`

### 4.4 Verify iSCSI Target Server virtual disk operational status

**Command:** `Get-IscsiVirtualDisk | Select-Object Path, DiskType, SizeBytes, OperationalStatus`

**What it checks and variables to look for:**

- **Path**: Must match physical path (e.g., `C:\iSCSIDisks\DataDisk01.vhdx`)
- **OperationalStatus**: Must be `0` (indicating normal/healthy operation)

<!-- Created by: Gergő Téringer, 2026 -->