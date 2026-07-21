<!-- 
---
title: "Windows Server DHCPv6 Configuration"
author: "Gergő Téringer"
---
 -->
# Windows Server DHCPv6 Configuration

> [!NOTE]
> Unlike IPv4, DHCPv6 operates slightly differently in a Windows environment. The most critical differences are that IPv6 does not hand out Default Gateways via DHCP (this is handled by Router Advertisements via NDP) and Windows Server does not support native DHCP Failover relationships for IPv6 scopes.

## 1. Installation & AD Authorization

Install the DHCP Server role and authorize it within Active Directory so it can begin serving clients on the network.

```powershell
# Install the DHCP role and management tools
Install-WindowsFeature -Name DHCP -IncludeManagementTools

# Authorize the DHCP server in Active Directory
# Note: You must be an Enterprise Admin to run this command successfully
Add-DhcpServerInDC

# Restart the DHCP service to ensure authorization takes effect
Restart-Service dhcpserver
```

## 2. Scope Configuration

Create the IPv6 scope and define its standard network options (DNS Servers, Domain Search List). Be sure to replace the variables with your environment's specific IPv6 prefixes and addresses.

```powershell
# Define scope variables
$ScopeName  = "RemoteNet_IPv6"
$Prefix     = "2001:db8:10::"
$StartIP    = "2001:db8:10::100"
$EndIP      = "2001:db8:10::200"
$DnsDomain  = "example.net"
$DnsServer  = "2001:db8:10::5"

# Create the new active scope
$newScope = Add-DhcpServerv6Scope -Name $ScopeName -Prefix $Prefix -StartRange $StartIP -EndRange $EndIP -State Active -PassThru

# Configure scope options (023 DNS Servers, 024 Domain Search List)
Set-DhcpServerv6OptionValue -Prefix $Prefix -DnsServer $DnsServer -DomainSearchList $DnsDomain
```

> [!WARNING]  
> Do not attempt to look for a "Router" or "Gateway" option in DHCPv6. IPv6 clients rely exclusively on the **Router Advertisement (RA)** packets sent by your network's gateway router to determine their default route.

## 3. High Availability (Scope Preference)

Because the `Add-DhcpServerv4Failover` command and its underlying protocol do not exist for IPv6 in Windows Server, you must rely on standard RFC-compliant DHCPv6 redundancy. The easiest method is an Active/Passive setup utilizing the **Preference** attribute.

When a client broadcasts for an IPv6 address, it waits to hear from multiple servers. It will always select the lease from the server broadcasting the highest preference value.

```powershell
# On your PRIMARY DHCPv6 Server, set the preference to maximum (255)
Set-DhcpServerv6Scope -Prefix $Prefix -Preference 255

# ---------------------------------------------------------
# On your SECONDARY DHCPv6 Server, create the exact same scope
# but set the preference to a lower value (e.g., 0)
# ---------------------------------------------------------
# $newScope = Add-DhcpServerv6Scope -Name $ScopeName -Prefix $Prefix -StartRange $StartIP -EndRange $EndIP -State Active
# Set-DhcpServerv6OptionValue -Prefix $Prefix -DnsServer $DnsServer -DomainSearchList $DnsDomain
# Set-DhcpServerv6Scope -Prefix $Prefix -Preference 0
```

## 4. Verification

Use the following commands to confirm your DHCPv6 server is healthy and options are applied correctly:

```powershell
# Verify the DHCP server is authorized in AD
Get-DhcpServerInDC

# List all configured IPv6 scopes and their active states
Get-DhcpServerv6Scope

# Verify the configured options for your specific IPv6 prefix
Get-DhcpServerv6OptionValue -Prefix "2001:db8:10::"
```

## 5. Troubleshooting

If IPv6 clients are not receiving IPs, use these commands to diagnose the DHCPv6 engine.

### 5.1 Verifying DHCPv6 Client Leases

**Command:** `Get-DhcpServerv6Lease -Prefix "2001:db8:10::"`

**What it checks and variables to look for:**

- `ClientIPv6Address`: Confirms the specific address handed out to the client.
- `ClientDuid`: The DHCP Unique Identifier (DUID) of the client. Unlike IPv4 which relies heavily on MAC addresses, IPv6 relies on the `DUID` for reservations and tracking.
- If this returns empty, clients are either not requesting addresses, or multicast traffic (`ff02::1:2`) is being blocked on the local subnet.

### 5.2 Verifying Event Logs for Errors

**Command:** `Get-WinEvent -LogName "Microsoft-Windows-DHCP-Server/Operational" -MaxEvents 20 | Where-Object {$_.Message -match "v6"} | Select-Object TimeCreated, Id, Message`

**What it checks and variables to look for:**

- **Id**: Look for critical error IDs representing scope exhaustion or service failures.
- **Message**: Filters the operational log specifically for strings containing `v6` to isolate IPv6 engine events from standard IPv4 noise.

### 5.3 Verifying Firewall Rules for DHCPv6

**Command:** `Get-NetFirewallRule -DisplayGroup "DHCP Server" | Select-Object Name, Enabled, Action`

**What it checks and variables to look for:**

- **Enabled**: Must be set to `True`.
- **Action**: Must be set to `Allow`.
- Specifically ensure that the inbound rules for **UDP Port 547** (DHCPv6 Server) are active, as IPv6 uses entirely different ports than IPv4 (which uses `67` and `68`).

<!-- Created by: Gergő Téringer, 2026 -->