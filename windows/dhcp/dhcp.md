# DHCP

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

```powershell
Add-DHCPServerInDC

$newScope = Add-DhcpServerv4Scope -Name RemoteNet0 `
  -StartRange 10.0.1.20 `
  -EndRange 10.0.1.200 `
  -SubnetMask 255.255.255.0 `
  -State Active `
  -PassThru

Set-DhcpServerv4OptionValue -ScopeID $newScope.ScopeId `
  -DnsDomain eaxmple.net `
  -DnsServer 10.0.0.5 `
  -Router 10.0.1.1

Set-DhcpServerv4Scope -ScopeId $newScope.ScopeID -State Active

```