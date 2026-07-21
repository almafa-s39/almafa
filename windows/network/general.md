<!-- 
---
title: "Windows Initial Configuration Guide"
author: "Gergő Téringer"
---
-->
# Windows Initial Configuration Guide

This guide outlines the standard operating procedures for initial Windows Server/Client configuration, split into local computer settings and Active Directory Group Policy Objects (GPOs).

## 1. Local Computer Configurations

Execute these commands in an elevated PowerShell console.

### 1.1 Time Settings

Configure the system time zone and manual date/time.

```powershell
# Set the time zone and date
Set-TimeZone -Id "<TIMEZONE_ID>" # e.g., "Central Europe Standard Time"
Set-Date -Date "<DATE>"          # e.g., "2026-06-17 12:14"
```

**Verification:**

```powershell
# Verify time zone and current system time
Get-TimeZone
Get-Date
```

### 1.2 Networking Settings

Configure static IPv4 and IPv6 addressing, gateways, and DNS servers using `netsh`.

```powershell
# IPv4 Configuration
netsh int ipv4 set address name="<INTERFACE>" static <ADDRESS> <SUBNET> <GW>
netsh int ipv4 set dns name="<INTERFACE>" static <DNS>

# IPv6 Configuration
netsh int ipv6 set address name="<INTERFACE>" <ADDRESS>/<PREFIX_LENGTH>
netsh int ipv6 add route ::/0 "<INTERFACE>" <GW>
netsh int ipv6 set dns name="<INTERFACE>" static <DNS>
```

**Verification:**

```powershell
# Verify overall IP and DNS configuration
ipconfig /all

# Verify routing tables
Get-NetRoute -InterfaceAlias "<INTERFACE>"
```

### 1.3 Allow ICMP (Ping)

Enable the built-in firewall rules to allow inbound Echo Requests for both IPv4 and IPv6.

```powershell
# Enable the default ICMP echo request rules
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
Enable-NetFirewallRule -Name "FPS-ICMP6-ERQ-In"

Set-NetFirewallProfile Private,Public,Domain -Enabled false
```

**Verification:**

```powershell
# Check the enabled state of the rules
Get-NetFirewallRule -Name "FPS-ICMP4-ERQ-In", "FPS-ICMP6-ERQ-In" | Select-Object Name, DisplayName, Enabled
```

### 1.4 Hostname Configuration

Rename the computer and reboot.

```powershell
Rename-Computer -NewName "<NEW_NAME>"
Restart-Computer
```

**Verification:** *(Run after reboot)*

```powershell
# Verify the new hostname applied successfully
$env:COMPUTERNAME
```

### 1.5 Domain Join

Join the computer to an Active Directory domain and reboot.

```powershell
Add-Computer -DomainName "<DOMAIN_NAME>" -Restart
```

**Verification:** *(Run after reboot)*

```powershell
# Verify domain membership
(Get-CimInstance Win32_ComputerSystem).Domain
```

---

## 2. Active Directory Domain Configurations (GPOs)

Configure the following settings within the Group Policy Management Console (`gpmc.msc`).

### 2.1 Quality of Life & Automation

These settings disable locking, remove the CTRL+ALT+DEL requirement, and skip the Windows first sign-in animation.

| Policy Objective | GPO Path | Setting |
| :--- | :--- | :--- |
| **Prevent screen saver** | `User Configuration > Policies > Administrative Templates > Control Panel > Personalization > Enable screen saver` | **Disabled** |
| **Remove screen saver password** | `User Configuration > Policies > Administrative Templates > Control Panel > Personalization > Password protect the screen saver` | **Disabled** |
| **Prevent machine lock** | `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Interactive logon: Machine inactivity limit` | **0** |
| **Disable CTRL+ALT+DEL** | `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Interactive logon: Do not require CTRL+ALT+DEL` | **Enabled** |
| **Disable sign-in animation** | `Computer Configuration > Policies > Administrative Templates > System > Logon > Show first sign-in animation` | **Disabled** |

### 2.2 Network & Firewall Policies

These settings manage the Windows Defender Firewall states across the domain and allow ping requests globally.

#### Allow ICMPv4 & ICMPv6 Inbound

* **Path:** `Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Windows Defender Firewall with Advanced Security > Inbound Rules`
* **Action:** Create two new Custom Rules (one for ICMPv4, one for ICMPv6) allowing **Echo Request** traffic from Any IP to Any IP.

#### Disable Windows Firewall

* **Path:** `Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Windows Defender Firewall with Advanced Security`
* **Action:** Right-click the root node -> **Properties**. Set the **Firewall state** to **Off** via the dropdown menus for the Domain Profile, Private Profile, and Public Profile.

<!-- Created by: Gergő Téringer, 2026 -->