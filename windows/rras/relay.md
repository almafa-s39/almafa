<!-- 
---
title: "RRAS DHCP Relay Configuration"
author: "Gergő Téringer"
---
-->
# RRAS DHCP Relay Configuration

## Configuration

This guide assumes you already have **RRAS installed** and configured for IPv4 routing.

To configure the DHCP Relay Agent, open an elevated PowerShell or Command Prompt window and execute the following commands. Be sure to replace `<DHCP_Server_IP>` with the IP address of your central DHCP server, and `<InterfaceName>` with the name of the network interface that will listen for DHCP broadcast requests from clients.

```powershell
# 1. Install the DHCP Relay Agent routing protocol
netsh routing ip relay install

# 2. Add the target DHCP server IP address where requests will be forwarded
netsh routing ip relay add dhcpserver <DHCP_Server_IP>

# 3. Add the interface that will listen for DHCP client broadcasts
netsh routing ip relay add interface name="<InterfaceName>"

# 4. Enable relay mode on the interface and configure hop/time thresholds
netsh routing ip relay set interface name="<InterfaceName>" relaymode=enable maxhop=4 minsecs=1
```

## Verification

To verify that the DHCP Relay Agent is correctly installed, bound to the proper interfaces, and forwarding traffic, you can use the following verification commands:

```powershell
# Show the global DHCP Relay Agent configuration
netsh routing ip relay show global

# Verify the list of configured target DHCP server IPs
netsh routing ip relay show dhcpserver

# Check the configuration, operational state, and statistics of the DHCP Relay interface
netsh routing ip relay show interface name="<InterfaceName>"
```

<!-- Created by: Gergő Téringer, 2026 -->