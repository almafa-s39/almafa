<!-- 
---
title: "NLB"
author: "Gergő Téringer"
---
 -->
# NLB

This document provides administrative procedures for deploying, managing, and troubleshooting Windows Network Load Balancing (NLB) clusters on Windows Server 2025 and modern Windows environments. It covers cluster operational modes, port rules, node management, and step-by-step procedures using both PowerShell cmdlets and Graphical User Interface (GUI) management consoles.

> [!NOTE]
> Network Load Balancing (NLB) enhances the availability and scalability of stateless Internet Protocol (IP) services—such as web servers (IIS), FTP, and VPN gateways—by distributing client traffic across up to 32 cluster nodes under a shared Virtual IP (VIP) address.

## 1. NLB Cluster Architecture and Operational Modes

When designing an NLB cluster, selecting the appropriate cluster operation mode is critical for proper network switch forwarding and inter-node communication.

- **Unicast Mode**: The cluster assigns a single shared virtual MAC address to all cluster adapters, overriding their physical MAC addresses. Because all nodes share the same MAC address, nodes **cannot** communicate with each other using cluster adapters without dedicated secondary network interfaces.
- **Multicast Mode**: Assigns a multicast MAC address to the virtual IP while preserving the original physical MAC address on each node's network adapter. This allows nodes to communicate with each other over a single NIC, but requires static ARP entries on upstream routers.
- **IGMP Multicast Mode**: A variation of Multicast mode that enables Internet Group Management Protocol (IGMP) joining messages to prevent switch flooding when IGMP snooping is enabled on physical network switches.

> [!IMPORTANT]
> To prevent network switch port flooding in Multicast modes, upstream network switches must support IGMP snooping or be configured with static ARP mappings binding the Virtual IP to the virtual Multicast MAC address.

Before lists place a blank line!

- **Filtering Mode - Multiple Host**: Distributes traffic across multiple nodes according to set load weightings.
- **Affinity - None**: Routes successive requests from the same client IP address to any available node (ideal for completely stateless applications).
- **Affinity - Single**: Binds successive requests from the same client IP address to the same cluster node using session persistence (ideal for web applications holding session state).
- **Affinity - Class C**: Binds requests originating from the same Class C IPv4 subnet (`/24`) to the same node.

## 2. NLB Cluster Provisioning

Provisioning an NLB cluster requires installing the NLB feature on all participating member nodes, creating a virtual cluster IP, and configuring port filtering rules.

### 2.1 Provisioning an NLB Cluster via PowerShell

```PowerShell
# Install NLB Feature and Management Tools on local node
Install-WindowsFeature -Name NLB, RSAT-NLB -IncludeManagementTools

# Create a new NLB Cluster on the primary node in Multicast mode
New-NlbCluster -InterfaceName "Ethernet" `
               -ClusterIp "192.168.10.100" `
               -ClusterSubnetMask "255.255.255.0" `
               -OperationMode Multicast `
               -NodeName "NLB01"

# Add a Port Rule for Web Traffic (HTTP/HTTPS) with Single Affinity
Add-NlbClusterPortRule -InterfaceName "Ethernet" `
                       -StartPort 80 `
                       -EndPort 443 `
                       -Protocol TCP `
                       -Mode Multiple `
                       -Affinity Single

# Add a secondary node to the existing NLB cluster
Add-NlbClusterNode -InterfaceName "Ethernet" `
                   -NewNodeName "NLB02" `
                   -NewNodeInterface "Ethernet" `
                   -ClusterIp "192.168.10.100"
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature -Name NLB, RSAT-NLB`: Installs the Network Load Balancing driver and RSAT management tools.
- `New-NlbCluster`: Initializes a new NLB cluster entity on the target server interface.
- `-ClusterIp "192.168.10.100"`: Defines the shared Virtual IP (VIP) that clients contact to access the service.
- `-OperationMode Multicast`: Configures cluster communication using Multicast MAC addressing to allow host-to-host traffic over a single network interface.
- `Add-NlbClusterPortRule`: Defines traffic routing logic for specific port ranges.
- `-StartPort 80 -EndPort 443`: Restricts load balancing logic to HTTP and HTTPS web traffic.
- `-Affinity Single`: Enforces client IP session stickiness to the same cluster host.
- `Add-NlbClusterNode`: Binds an additional host node to the running NLB cluster topology.

### 2.2 Provisioning an NLB Cluster via GUI

1. Open **Network Load Balancing Manager** (`nlbmgr.exe`) from the Start menu or Run dialog.
2. Right-click **Network Load Balancing Clusters** and select **New Cluster**.
3. In the **Host** field, enter the primary node FQDN (e.g., `NLB01.corp.contoso.com`) and click **Connect**.
4. Select the network interface to use for cluster traffic and click **Next**.
5. Assign a host priority (ID) and default initial host state (**Started**), then click **Next**.
6. Click **Add** to specify the shared **Cluster IP Address** (e.g., `192.168.10.100`) and subnet mask, then click **Next**.
7. Enter the **Full Internet Name** (e.g., `web.corp.contoso.com`) and select the **Cluster operation mode** (**Unicast**, **Multicast**, or **IGMP Multicast**). Click **Next**.
8. Edit or define **Port Rules** (e.g., Ports `80` to `443`, Protocol `TCP`, Filtering mode `Multiple host`, Affinity `Single`).
9. Click **Finish** to provision the cluster.
10. To add additional nodes: Right-click the newly created cluster, select **Add Host To Cluster**, connect to the secondary host (`NLB02`), and complete the wizard.

## 3. Node Control and Maintenance Operations

When performing software updates, security patching, or maintenance on individual cluster hosts, nodes should be cleanly drained to prevent dropping active client sessions.

> [!TIP]
> Use the **Drainstop** action instead of an immediate **Stop**. Drainstop prevents the node from accepting new client connections while allowing existing active sessions to complete naturally before stopping cluster services.

### 3.1 Managing Cluster Nodes via PowerShell

```PowerShell
# Gracefully drain existing sessions and stop NLB traffic on a maintenance host
Stop-NlbClusterNode -HostName "NLB01" -Drain

# Immediately suspend NLB cluster operations on a host without draining
Stop-NlbClusterNode -HostName "NLB01"

# Resume and start NLB cluster operations on a node after maintenance
Start-NlbClusterNode -HostName "NLB01"
```

**Command Breakdown & Explanation:**

- `Stop-NlbClusterNode`: Halts NLB network processing on the designated node.
- `-Drain`: Instructs the node to finish handling current active client connections while rejecting new inbound requests prior to taking the host offline.
- `Start-NlbClusterNode`: Re-engages the host node into the active NLB cluster ring, initiating state convergence.

### 3.2 Managing Cluster Nodes via GUI

1. Open **Network Load Balancing Manager** (`nlbmgr.exe`).
2. Expand the cluster tree node to display active member hosts.
3. Right-click the target node (e.g., `NLB01`) navigate to **Control Host**, and select the desired action:
   - Select **Drainstop** for graceful maintenance shutdown.
   - Select **Stop** for immediate shutdown.
   - Select **Start** to bring the node back into converged production status.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate NLB cluster state, node convergence, virtual IP assignments, and port rule behavior on Windows 11 or Windows Server 2025 using elevated diagnostic cmdlets.

### 4.1 Verify NLB cluster state and node convergence status

**Command:** `Get-NlbClusterNode | Select-Object HostName, State, InterfaceName`

**What it checks and variables to look for:**

- **HostName**: Must list all participating cluster hosts (`NLB01`, `NLB02`)
- **State**: Must be `Converged` (A state of `Draining` indicates active maintenance; `Disconnected` or `Stopped` indicates failure or manual stoppage)

### 4.2 Verify virtual IP and adapter bindings

**Command:** `Get-NlbCluster | Select-Object ClusterName, ClusterIPAddress, OperationMode`

**What it checks and variables to look for:**

- **ClusterIPAddress**: Must match the configured Virtual IP (`192.168.10.100`)
- **OperationMode**: Must show `Multicast` or `Unicast` matching switch infrastructure requirements

### 4.3 Verify port rule configuration and affinity mode

**Command:** `Get-NlbClusterPortRule | Select-Object StartPort, EndPort, Protocol, Mode, Affinity`

**What it checks and variables to look for:**

- **StartPort**: Must show `80`
- **EndPort**: Must show `443`
- **Protocol**: Must be `TCP`
- **Affinity**: Must show `Single`

<!-- Created by: Gergő Téringer, 2026 -->