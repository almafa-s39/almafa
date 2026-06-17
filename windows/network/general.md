# General configurations

## Per computer

### Time settings

```powershell
Get-TimeZone
Set-TimeZone "[TIMEZONE]"
Set-Date "[DATE]" # [DATE] = "2026.06.17 12:14"
```

### Networking settings

```powershell
ipconfig /all
netsh int ipv4 set add [INTERFACE] static [ADDRESS] [SUBNET] [GW]
netsh int ipv4 set dns [INTERFACE] static [DNS]
netsh int ipv6 set add [INTERFACE] [ADDRESS]/[NETMASK]
netsh int ipv6 add route ::/0 [INTERFACE] [GW]
netsh int ipv6 set dns [INTERFACE] static [DNS]
```

### ALLOW ICMP

```powershell
Get-NetFirewallRule | ? { $_.displayName -like "*ICMPv4-In*" } | Select-Object Name, DisplayName, Enabled

# Parsed names from the output
Enable-NetFirewallRule "FPS-ICMP4-ERQ-In"
Enable-NetFirewallRule "FPS-ICMP6-ERQ-In"
```

### Hostname

```powershell
Rename-Computer [NEW-NAME]
Restart-Computer
```

### Add copmuter to the domain

```powershell
Add-Computer -DomainName [DOMAIN-NAME]
Restart-Computer
```

## Per domain (GPOs)

### Prevent lock

`User Configuration > Policies > Administrative Templates > Control Panel > Personalization > Enable screen saver` > `Disabled`
`User Configuration > Policies > Administrative Templates > Control Panel > Personalization > Password protect the screen saver` > `Disabled`
`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Interactive logon: Machine inactivity limit` > `0`

### Disable CTRL+ALT+DEL

`Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options > Do not require CTRL+ALT+DEL` > `Enabled`

### Disable first animation login

`Computer Configuration > Policies > Administrative Templates > System > Logon > Show first sign-in animation` > `Disabled`

### Enable ICMPv4, ICMPv6

`Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Windows Defender Firewall with Advanced Security > Inbound Rules` > Create two custom rule

### Disable firewall

`Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall with Advanced Security > Windows Defender Firewall with Advanced Security` > Right-click node > Properties, Choose Domain-, Private-, Public Profile, and set Firewall state Off from the dropdown.
