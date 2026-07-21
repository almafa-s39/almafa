<!-- 
---
title: "Windows Server BGP Configuration"
author: "Gergő Téringer"
---
-->
# Windows Server BGP Configuration

## 1. Install Routing and Remote Access (RRAS)

First, install the required Windows features and initialize the RRAS service specifically for routing.

```powershell
# Install the Routing role and management tools
Install-WindowsFeature -Name Routing -IncludeManagementTools

# A restart is required to finish the installation
Restart-Computer 

# After the reboot, initialize RRAS for LAN routing only
Install-RemoteAccess -VpnType RoutingOnly
```

## 2. Configure BGP

Once RRAS is installed and running, initialize the local BGP router, enable IPv6 address family support, establish peering, and inject routes into the BGP table.

```powershell
# Initialize the local BGP router with your Router ID and Autonomous System Number
Add-BgpRouter -BgpIdentifier <RID> -LocalASN <LOCAL_AS> 

# Enable IPv6 routing support for the BGP process
Set-BgpRouter -BgpIdentifier <RID> -IPv6Routing Enabled

# Add a BGP peer (neighbor)
Add-BgpPeer -PeerName "<STRING>" -LocalIPAddress <LOCAL_PEER_ADDRESS> -PeerIPAddress <PEER_ADDRESS> -LocalASN <LOCAL_AS> -PeerASN <PEER_AS>

# Advertise local networks into the BGP routing table
Add-BgpCustomRoute -Network <NETWORK_ADD>/<PREFIX_LENGTH>
```

## 3. Verification

Use the following cmdlets to verify your BGP configuration, check neighbor adjacency states, and inspect learned routes:

```powershell
# View the local BGP router configuration
Get-BgpRouter

# Check the status of BGP peers (look for ConnectivityStatus: Connected)
Get-BgpPeer

# View the BGP routing table (learned and locally originated routes)
Get-BgpRouteInformation
```

<!-- Created by: Gergő Téringer, 2026 -->