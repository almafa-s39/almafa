# General configurations

## Time settings

```powershell
Get-TimeZone
Set-TimeZone "[TIMEZONE]"
Set-Date "[DATE]" # [DATE] = "2026.06.17 12:14"
```

## Networking settings

```powershell
ipconfig /all
netsh int ipv4 set add [INTERFACE] static [ADDRESS] [SUBNET] [GW]
netsh int ipv4 set dns [INTERFACE] static [DNS]
netsh int ipv6 set add [INTERFACE] [ADDRESS]/[NETMASK]
netsh int ipv6 add route ::/0 [INTERFACE] [GW]
netsh int ipv6 set dns [INTERFACE] static [DNS]
```

## Hostname

```powershell
Rename-Computer [NEW-NAME]
Restart-Computer
```
