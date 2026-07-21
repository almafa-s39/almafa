<!-- 
---
title: "DFS"
author: "Gergő Téringer"
---
-->
# DFS

This document details administrative procedures for deploying Active Directory-integrated Distributed File System Namespaces (DFS-N) and DFS Replication (DFS-R) on Windows Server 2025 and Windows 11 environments. It provides implementation steps using both PowerShell cmdlets and Graphical User Interface (GUI) management consoles.

> [!NOTE]
> DFS Namespaces group geographically scattered network shares into a single virtual namespace (`\\domain.com\namespace`). DFS Replication synchronizes folder contents across multiple servers over LAN or WAN connections using Remote Differential Compression (RDC).

## 1. Active Directory-Integrated DFS Namespaces (DFS-N) Provisioning

An Active Directory Domain-based namespace (DomainV2) stores its namespace metadata within Active Directory Domain Services (AD DS). This architecture ensures high availability without requiring a failover cluster, as any domain controller or designated namespace server can respond to client referrals.

> [!IMPORTANT]
> Domain-based namespaces must be created on NTFS or ReFS volumes. Shared permissions and NTFS File System Access Control Lists (ACLs) must be configured consistently across all target servers.

### 1.1 Configuring DFS Namespaces via PowerShell

```PowerShell
# Install DFS Management Tools and Role Services
Install-WindowsFeature -Name FS-DFS-Namespace, RSAT-DFS-Mgmt-Con -IncludeManagementTools

# Create a Domain-based DFS Root Namespace (DomainV2)
New-DfsnRoot -Path "\\corp.contoso.com\SharedData" `
             -TargetPath "\\FS01.corp.contoso.com\SharedData" `
             -Type DomainV2

# Add a DFS Folder within the Namespace
New-DfsnFolder -Path "\\corp.contoso.com\SharedData\Finance" `
               -TargetPath "\\FS01.corp.contoso.com\FinanceShare"

# Add a secondary Target Server to the DFS Folder for High Availability
New-DfsnFolderTarget -Path "\\corp.contoso.com\SharedData\Finance" `
                     -TargetPath "\\FS02.corp.contoso.com\FinanceShare"
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name FS-DFS-Namespace, RSAT-DFS-Mgmt-Con`: Installs the DFS Namespace role service along with the graphical management tools.
- `New-DfsnRoot`: Provisions a new DFS namespace root.
- `-Path "\\corp.contoso.com\SharedData"`: Defines the logical Active Directory UNC path accessible to domain clients.
- `-TargetPath "\\FS01.corp.contoso.com\SharedData"`: Specifies the actual physical network share hosting the primary namespace root structure.
- `-Type DomainV2`: Specifies Windows Server 2008/2012+ Active Directory-integrated mode for enterprise scalability and access-based enumeration support.
- `New-DfsnFolder`: Creates a virtual folder inside the namespace root.
- `New-DfsnFolderTarget`: Binds an additional physical share location to an existing virtual DFS folder, enabling referral load balancing.

### 1.2 Configuring DFS Namespaces via GUI

1. Open **DFS Management** (`dfsmgmt.msc`) from the Start menu or Run dialog.
2. Right-click **Namespaces** in the left console tree and select **New Namespace**.
3. In the **Server** wizard page, enter the server FQDN (e.g., `FS01.corp.contoso.com`) and click **Next**.
4. Enter the **Namespace Name** (e.g., `SharedData`) and click **Next**.
5. Select **Domain-based namespace** and ensure **Enable Windows Server 2008 mode** is checked. Click **Next**.
6. Review settings and click **Create**, then click **Close**.
7. Expand **Namespaces**, right-click `\\corp.contoso.com\SharedData`, and select **New Folder**.
8. Enter the **Folder name** (e.g., `Finance`), click **Add**, and enter the target share path (`\\FS01.corp.contoso.com\FinanceShare`).
9. Click **OK** twice to complete folder creation.

## 2. DFS Replication (DFS-R) Configuration

DFS Replication uses a multi-master replication engine to automatically synchronize files between folder targets. It compresses changes using Remote Differential Compression (RDC) so that only modified data blocks travel over the network.

> [!CAUTION]
> When setting up a new DFS Replication group, exactly one server must be designated as the **Primary Member**. The primary member's content overwrites any existing content on secondary members during initial synchronization.

### 2.1 Configuring DFS Replication via PowerShell

```PowerShell
# Install DFS Replication Service
Install-WindowsFeature -Name FS-DFS-Replication

# Create a Replication Group
New-DfsReplicationGroup -GroupName "RG-Finance"

# Add Member Servers to the Replication Group
Add-DfsReplicationGroupMember -GroupName "RG-Finance" `
                              -ComputerName "FS01.corp.contoso.com", "FS02.corp.contoso.com"

# Define the Replicated Folder
Add-DfsReplicatedFolder -GroupName "RG-Finance" `
                        -FolderName "FinanceData"

# Configure Membership Settings, Local Folder Paths, and Primary Member status
Set-DfsrMembership -GroupName "RG-Finance" `
                   -FolderName "FinanceData" `
                   -ComputerName "FS01.corp.contoso.com" `
                   -ContentPath "C:\Shares\FinanceShare" `
                   -PrimaryMember $true

Set-DfsrMembership -GroupName "RG-Finance" `
                   -FolderName "FinanceData" `
                   -ComputerName "FS02.corp.contoso.com" `
                   -ContentPath "C:\Shares\FinanceShare"

# Create Full Mesh Replication Connection Topology
Add-DfsrConnection -GroupName "RG-Finance" `
                   -SourceComputerName "FS01.corp.contoso.com" `
                   -DestinationComputerName "FS02.corp.contoso.com"
```

**Command Breakdown & Explanation:**

- `New-DfsReplicationGroup`: Initializes a logical container for managing file replication between servers.
- `Add-DfsReplicationGroupMember`: Registers specific domain member servers into the replication group.
- `Add-DfsReplicatedFolder`: Defines the logical name of the synchronized folder across the group.
- `Set-DfsrMembership`: Maps the replicated folder to a physical local drive path (`ContentPath`) on each participating server.
- `-PrimaryMember $true`: Designates the authoritative source server during initial sync to prevent data loss.
- `Add-DfsrConnection`: Configures bi-directional replication links between source and destination endpoints.

### 2.2 Configuring DFS Replication via GUI

1. Open **DFS Management** (`dfsmgmt.msc`).
2. Right-click **Replication** in the left console tree and select **New Replication Group**.
3. Select **Multipurpose replication group** and click **Next**.
4. Enter the **Name of replication group** (e.g., `RG-Finance`) and click **Next**.
5. Click **Add**, type the names of member servers (`FS01`, `FS02`), and click **Next**.
6. Select the topology type (recommended: **Full mesh**) and click **Next**.
7. Specify replication schedule and bandwidth allowance (e.g., **Full bandwidth**), then click **Next**.
8. Select the **Primary member** (e.g., `FS01`) responsible for initial sync and click **Next**.
9. Assign the local folder path (`C:\Shares\FinanceShare`) on each server and click **Next**.
10. Review configurations, click **Create**, and then click **Close**.

## 3. Verification and Troubleshooting

> [!NOTE]
> Execute these verification cmdlets in an elevated PowerShell session on Windows 11 or Windows Server 2025 to validate namespace referrals, replication health, and sync backlogs.

### 3.1 Verify DFS Namespace health and root target status

**Command:** `Get-DfsnRoot -Path "\\corp.contoso.com\SharedData" | Select-Object Path, Type, State`

**What it checks and variables to look for:**

- **Path**: Must be `\\corp.contoso.com\SharedData`
- **Type**: Must be `DomainV2`
- **State**: Must be `Online`

### 3.2 Verify DFS Folder target availability and referral state

**Command:** `Get-DfsnFolderTarget -Path "\\corp.contoso.com\SharedData\Finance" | Select-Object Path, TargetPath, State, Referrals`

**What it checks and variables to look for:**

- **TargetPath**: Must list all configured endpoints (`\\FS01.corp.contoso.com\FinanceShare`, `\\FS02.corp.contoso.com\FinanceShare`)
- **State**: Must be `Online`
- **Referrals**: Must be `Enabled`

### 3.3 Verify DFS Replication Group membership and local folder bindings

**Command:** `Get-DfsrMembership -GroupName "RG-Finance" | Select-Object ComputerName, FolderName, ContentPath, PrimaryMember`

**What it checks and variables to look for:**

- **ComputerName**: Must list participating servers (`FS01`, `FS02`)
- **ContentPath**: Must point to valid local physical directories (e.g., `C:\Shares\FinanceShare`)
- **PrimaryMember**: Must show `True` on initial source server and `False` on remaining members

### 3.4 Verify DFS Replication backlog and pending sync items between nodes

**Command:** `Get-DfsrBacklog -GroupName "RG-Finance" -FolderName "FinanceData" -SourceComputerName "FS01.corp.contoso.com" -DestinationComputerName "FS02.corp.contoso.com"`

**What it checks and variables to look for:**

- **BacklogCount**: Must be `0` (A value higher than `0` indicates pending items actively replicating or stalled sync)

<!-- Created by: Gergő Téringer, 2026 -->