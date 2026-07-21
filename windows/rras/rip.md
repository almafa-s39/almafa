<!-- 
---
title: "RRAS RIP Configuration"
author: "Gergő Téringer"
---
 -->
# RRAS RIP Configuration

## Prerequisites

This guide assumes you already have **RRAS installed and configured** for LAN/WAN routing.

*Note: Unlike BGP, Windows Server does not include native PowerShell cmdlets (e.g., `Add-RipRouter`) for the legacy RIP protocol. The standard supported method for command-line configuration in Windows is using `netsh` routing contexts, which are executed directly within your PowerShell scripts.*

## Configuration

Open an elevated PowerShell console and run the following commands. Be sure to replace the interface names with the actual names of the adapters in your topology.

```powershell
# 1. Install the RIP routing protocol inside the RRAS IPv4 routing context
netsh routing ip rip install

# 2. Add the participating interfaces to the RIP protocol
# Replace "Ethernet1" and "Ethernet2" with your actual routing interfaces
netsh routing ip rip add interface "Ethernet1"
netsh routing ip rip add interface "Ethernet2"

# 3. Configure the interfaces for RIPv2
# This disables RIPv1, disables authentication, and enables standard broadcasts
netsh routing ip rip set interface "Ethernet1" auth=none bcast=yes unicast=no ripv1=no
netsh routing ip rip set interface "Ethernet2" auth=none bcast=yes unicast=no ripv1=no

# 4. (Optional) Configure Unicast neighbors instead of Broadcasts
# If your topology requires unicast updates, swap bcast/unicast and add the peer IP
# netsh routing ip rip set interface "Ethernet1" auth=none bcast=no unicast=yes ripv1=no
# netsh routing ip rip add peer "Ethernet1" 10.0.0.2

# 5. Restart the Remote Access service to apply and initialize the routing table
Restart-Service RemoteAccess
```

## Verification

To verify that RIP is running, interfaces are active, and routes are being learned, you can use the following commands in PowerShell:

```powershell
# Show global RIP statistics
netsh routing ip rip show global

# Show RIP interface configurations and states
netsh routing ip rip show interface

# Show the learned RIP peer neighbors
netsh routing ip rip show peer

# Check the Windows Server routing table to verify learned routes
Get-NetRoute | Where-Object { $_.RouteMetric -ne 256 } | Sort-Object DestinationPrefix
```

<!-- Created by: Gergő Téringer, 2026 -->