<!-- 
---
title: "Windows Server DHCP Configuration & Failover"
author: "Gergő Téringer"
---
-->
# Windows Server DHCP Configuration & Failover

## 1. Installation & AD Authorization

Install the DHCP Server role and authorize it within Active Directory so it can begin serving clients.

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

Create the IPv4 scope and define its standard network options (DNS, Gateway, Domain). Be sure to replace the variables with your environment's specific values.

```powershell
# Define scope variables
$ScopeName  = "RemoteNet0"
$StartIP    = "10.0.1.20"
$EndIP      = "10.0.1.200"
$SubnetMask = "255.255.255.0"
$DnsDomain  = "example.net"
$DnsServer  = "10.0.0.5"
$Router     = "10.0.1.1"

# Create the new active scope
$newScope = Add-DhcpServerv4Scope -Name $ScopeName -StartRange $StartIP -EndRange $EndIP -SubnetMask $SubnetMask -State Active -PassThru

# Configure scope options (003 Router, 006 DNS Servers, 015 DNS Domain Name)
Set-DhcpServerv4OptionValue -ScopeID $newScope.ScopeId -DnsDomain $DnsDomain -DnsServer $DnsServer -Router $Router

# Ensure the scope is fully active
Set-DhcpServerv4Scope -ScopeId $newScope.ScopeID -State Active
```

## 3. High Availability (Failover Configuration)

Configure DHCP Failover between two servers for redundancy. This guide uses **Load Balance** mode (Active/Active), but can be changed to **HotStandby** (Active/Passive) by modifying the `-Mode` parameter.

*Prerequisite: The partner server must already have the DHCP role installed and be authorized in AD.*

```powershell
# Define failover variables
$PartnerServer = "<PARTNER_FQDN_OR_IP>" # e.g., "dhcp02.example.net"
$FailoverName  = "Failover-$ScopeName"
$SharedSecret  = "<YOUR_SECURE_SECRET>"

# Create the failover relationship and add the scope to it
Add-DhcpServerv4Failover -ComputerName $env:COMPUTERNAME `
  -Name $FailoverName `
  -PartnerServer $PartnerServer `
  -ScopeId $newScope.ScopeId `
  -Mode LoadBalance `
  -LoadBalancePercent 50 `
  -SharedSecret $SharedSecret
```

## 4. Verification

Use the following commands to confirm your DHCP server and failover configuration are healthy:

```powershell
# Verify the DHCP server is authorized in AD
Get-DhcpServerInDC

# List all configured IPv4 scopes and their states
Get-DhcpServerv4Scope

# Verify the configured options for your specific scope
Get-DhcpServerv4OptionValue -ScopeId "10.0.1.0"

# Check the failover relationship status (State should be 'Normal')
Get-DhcpServerv4Failover -ComputerName $env:COMPUTERNAME
```

## 5. Troubleshooting

If clients are not receiving IPs or the failover state is degraded, use these commands to diagnose the issue:

```powershell
# 1. Check if the DHCP Server service is running
Get-Service dhcpserver

# 2. Check for active or pending client leases to see if any traffic is being processed
Get-DhcpServerv4Lease -ScopeId "10.0.1.0"

# 3. View recent DHCP Server event logs for critical errors or warnings
Get-WinEvent -LogName "Microsoft-Windows-DHCP-Server/Operational" -MaxEvents 20 | Select-Object TimeCreated, Id, Message

# 4. If failover is out of sync, force a replication from the primary to the partner
Invoke-DhcpServerv4FailoverReplication -ComputerName $env:COMPUTERNAME -Name $FailoverName

# 5. Ensure the Windows Firewall is allowing DHCP traffic (UDP 67/68)
Get-NetFirewallRule -DisplayGroup "DHCP Server" | Select-Object Name, Enabled, Action
```

<!-- Created by: Gergő Téringer, 2026 -->