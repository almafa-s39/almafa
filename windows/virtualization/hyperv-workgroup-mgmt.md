<!--
---
title: "ADCS SubCA with PowerShell"
author: "Simon Tamás"
---
-->

# ADCS SubCA with PowerShell

## 0. Used conventions

The following variables are used throughout the guide:

- Computers:
  - `TEST-SRV-1` (Management machine, DNS, WORKGROUP1)
  - `TEST-SRV-2` (Hyper-V Host, WORKGROUP2)
- DNS:
  - One forward zone with domain `hyperv.com`, records for both machines:
    - `TEST-SRV-1.hyperv.com -> 10.0.0.1`
    - `TEST-SRV-2.hyperv.com -> 10.0.0.2`

> [!IMPORTANT]
> DNS is crucial for this setup. Use FQDNs.

## 1. Common setup

The firewall rules used are only applied for the Domain/Private interfaces. Set the interfaces to be in one of these groups:

```powershell
Get-NetConnectionProfile
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
# You can also use the -InterfaceIndex option, which might be easier on the Hyper-V host as the vEthernet adapter name is very long
```

## 2. Hyper-V Host setup

```powershell
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

Enable-PSRemoting -Force
Enable-WSManCredSSP -Role Server -Force

Enable-NetFirewallRule -DisplayGroup "Hyper-V"
Enable-NetFirewallRule -DisplayGroup "Hyper-V Replica HTTP"   # only if you use replica
Enable-NetFirewallRule -DisplayGroup "Windows Management Instrumentation (WMI)"
Enable-NetFirewallRule -DisplayGroup "Remote Volume Management"  # optional, for disk mgmt

Add-LocalGroupMember -Group "Hyper-V Administrators" -Member Administrator
```

## 3. Management host setup

```powershell
Enable-PSRemoting -Force

# Use the exact name you will connect with
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "TEST-SRV-2.hyperv.com" -Concatenate -Force
Enable-WSManCredSSP -Role Client -DelegateComputer "TEST-SRV-2.hyperv.com" -Force
```

Open `gpedit.msc` Local Policy editor and set the following:

- Computer Configuration > Administrative Templates > System > Credentials Delegation > Allow delegating fresh credentials with NTLM-only server authentication:
  - Enabled
  - Add: `wsman/TEST-SRV-2.hyperv.com`

## Connect to Hyper-V

In Hyper-V Manager:

- Connect to Server
- Enter TEST-SRV-2.hyperv.com
- Tick Connect as another user
- Set User
- TEST-SRV-2\Administrator

<!-- Created by: Simon Tamás, 2026 -->
