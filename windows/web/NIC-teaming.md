<!-- 
---
title: "NIC-teaming"
author: "Gergő Téringer"
---
-->

# NIC teaming

This document details administrative procedures for configuring network interface redundancy and bandwidth aggregation on Windows Server 2025 and modern Windows environments. It covers traditional software Network Interface Card (NIC) Teaming (LBFO) as well as modern Hyper-V Switch Embedded Teaming (SET) using both PowerShell cmdlets and Graphical User Interface (GUI) workflows.

> [!NOTE]
> NIC Teaming binds multiple physical network adapters into a single logical network interface. This provides high availability through failover protection and increased throughput via traffic load balancing.

## 1. NIC Teaming Architecture and Modes

Choosing the correct teaming mode and load distribution algorithm depends on switch infrastructure capabilities and host role requirements.

> [!IMPORTANT]
> Traditional Load Balancing and Failover (LBFO) NIC Teaming (`NetLbfoTeam`) is supported for host-level OS networking on bare-metal servers. However, for Hyper-V environments on Windows Server 2022 and Windows Server 2025, Microsoft recommends **Switch Embedded Teaming (SET)** integrated directly into the Hyper-V Virtual Switch.

Before lists place a blank line!

- **Switch Independent Mode**: Requires no specialized switch configuration. Member NICs can connect to different physical switches, providing switch-level redundancy.
- **Static Teaming (IEEE 802.3ad draft)**: Requires physical switch configuration where all team member ports are manually configured into a static Link Aggregation Group (LAG).
- **LACP Mode (Link Aggregation Control Protocol - IEEE 802.1ax/802.3ad)**: Dynamically negotiates aggregation groups with physical switches supporting LACP.

Before lists place a blank line!

- **Dynamic Load Distribution**: Uses TCP port and IP address hashes combined with flowlet management to automatically balance inbound and outbound traffic. This is the recommended default for most workloads.
- **Hyper-V Port Load Distribution**: Maps virtual machine network adapters (vNICs) to specific physical team member adapters, making it ideal for Hyper-V hosts using traditional LBFO.
- **Address Hash Load Distribution**: Distributes outbound traffic based on source/destination IP, MAC, or transport port hashes.

## 2. Standard Software NIC Teaming (LBFO) Provisioning

Traditional LBFO teaming aggregates physical network interfaces at the operating system level, creating a unified `NetLbfoTeam` virtual adapter.

### 2.1 Configuring LBFO NIC Teaming via PowerShell

```PowerShell
# Provision a standard LBFO NIC Team in Switch Independent mode with Dynamic load distribution
New-NetLbfoTeam -Name "Team01" `
                -TeamMembers "Ethernet1", "Ethernet2" `
                -TeamingMode SwitchIndependent `
                -LoadBalancingAlgorithm Dynamic

# Modify an existing team to set a standby adapter for active-passive failover
Set-NetLbfoTeam -Name "Team01" `
                -StandbyAdapter "Ethernet2"
```

**Command Breakdown & Explanation:**

- `New-NetLbfoTeam`: Provisions a software-based LBFO team object.
- `-Name "Team01"`: Assigns the friendly name identifier for the logical teamed network interface.
- `-TeamMembers "Ethernet1", "Ethernet2"`: Specifies physical network adapter names bound to the team.
- `-TeamingMode SwitchIndependent`: Configures the team to operate without requiring LACP or static port channel configurations on upstream switches.
- `-LoadBalancingAlgorithm Dynamic`: Directs the driver to dynamically distribute outbound traffic flows across all active interfaces.
- `Set-NetLbfoTeam`: Updates configuration properties on an existing LBFO team instance.
- `-StandbyAdapter "Ethernet2"`: Designates a specific adapter to remain idle until an active interface fails.

### 2.2 Configuring LBFO NIC Teaming via GUI

1. Open **Server Manager**, navigate to **Local Server**.
2. Locate the **NIC Teaming** property field (defaults to *Disabled*) and click the status link to open `lbfoadmin.exe`.
3. In the **Tasks** menu under the **TEAMS** section, select **New Team**.
4. Enter a **Team name** (e.g., `Team01`).
5. Select the checkboxes next to the physical network adapters to include in the team.
6. Expand **Additional properties** to adjust operational modes:
   - **Teaming mode**: Select **Switch Independent**, **Static Teaming**, or **LACP**.
   - **Load balancing mode**: Select **Dynamic**, **Hyper-V Port**, or **Address Hash**.
   - **Standby adapter**: Select **None (all adapters Active)** or assign a designated standby interface.
7. Click **OK** to apply the configuration.

## 3. Switch Embedded Teaming (SET) for Hyper-V Environments

Switch Embedded Teaming (SET) integrates teaming functionality directly into the Hyper-V Virtual Switch (`VMSwitch`). It delivers higher throughput, lower CPU utilization, and compatibility with Remote Direct Memory Access (RDMA / RoCEv2).

> [!TIP]
> SET supports up to 8 physical network adapters inside a single virtual switch and requires all member physical interfaces to be identical in speed and configuration.

### 3.1 Configuring Switch Embedded Teaming via PowerShell

```PowerShell
# Create an Embedded Teaming Hyper-V Virtual Switch with two physical adapters
New-VMSwitch -Name "SET-VMSwitch" `
             -NetAdapterName "Ethernet3", "Ethernet4" `
             -EnableEmbeddedTeaming $true `
             -AllowManagementOS $true

# Set the SET team load balancing algorithm to Dynamic
Set-VMSwitchTeam -Name "SET-VMSwitch" `
                 -LoadBalancingAlgorithm Dynamic
```

**Command Breakdown & Explanation:**

- `New-VMSwitch`: Creates a new Hyper-V virtual switch instance.
- `-Name "SET-VMSwitch"`: Defines the friendly virtual switch name accessible to virtual machines and the host OS.
- `-NetAdapterName "Ethernet3", "Ethernet4"`: Binds multiple physical interfaces directly into the virtual switch.
- `-EnableEmbeddedTeaming $true`: Activates Switch Embedded Teaming (SET) logic directly inside the virtual switch core.
- `-AllowManagementOS $true`: Provisions a virtual network adapter on the host operating system for management communication.
- `Set-VMSwitchTeam`: Modifies SET-specific load balancing parameters.
- `-LoadBalancingAlgorithm Dynamic`: Sets the outbound hashing distribution algorithm across member physical ports.

### 3.2 Configuring Switch Embedded Teaming via GUI

1. Open **Hyper-V Manager** (`virtmgmt.msc`).
2. Click **Virtual Switch Manager** in the right Actions pane.
3. Select **New virtual network switch**, choose **External**, and click **Create Virtual Switch**.
4. Enter a **Name** (e.g., `SET-VMSwitch`).
5. Under **External network**, select the primary physical adapter.
6. Check **Allow management operating system to share this network adapter**.
7. Click **Apply** and **OK**.

> [!WARNING]
> Full Switch Embedded Teaming (SET) multi-adapter aggregation is configured primarily via PowerShell. The Hyper-V Manager GUI allows assigning single interfaces to external switches, but multi-NIC SET arrays require `New-VMSwitch -EnableEmbeddedTeaming $true` for initial binding.

## 4. Verification and Troubleshooting

> [!NOTE]
> Execute these verification cmdlets in an elevated PowerShell session on Windows 11 or Windows Server 2025 to validate team status, interface health, and load balancing configurations.

### 4.1 Verify LBFO team status and member interface health

**Command:** `Get-NetLbfoTeam | Select-Object Name, TeamingMode, LoadBalancingAlgorithm, Status`

**What it checks and variables to look for:**

- **Name**: Must be `Team01`
- **TeamingMode**: Must be `SwitchIndependent`, `Static`, or `LACP`
- **LoadBalancingAlgorithm**: Must be `Dynamic`
- **Status**: Must be `Up`

### 4.2 Verify LBFO team member interface operational state

**Command:** `Get-NetLbfoTeamMember | Select-Object Name, Team, AdministrativeMode, OperationalStatus`

**What it checks and variables to look for:**

- **Team**: Must match the assigned team name (`Team01`)
- **AdministrativeMode**: Must be `Active` or `Standby`
- **OperationalStatus**: Must be `Active`

### 4.3 Verify Hyper-V Switch Embedded Teaming status

**Command:** `Get-VMSwitchTeam -Name "SET-VMSwitch" | Select-Object Name, NetAdapterInterfaceDescription, LoadBalancingAlgorithm`

**What it checks and variables to look for:**

- **Name**: Must be `SET-VMSwitch`
- **NetAdapterInterfaceDescription**: Must list all participating physical network adapters
- **LoadBalancingAlgorithm**: Must be `Dynamic`

### 4.4 Verify physical network adapter link state

**Command:** `Get-NetAdapter -Name "Ethernet1", "Ethernet2" | Select-Object Name, Status, LinkSpeed`

**What it checks and variables to look for:**

- **Status**: Must be `Up`
- **LinkSpeed**: Must show valid link speed (e.g., `10 Gbps` or `1 Gbps`) matching peer adapters

<!-- Created by: Gergő Téringer, 2026 -->