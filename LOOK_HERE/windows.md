# PowerShell append to file

```powershell
$message = "Hello World"
$message | Out-File "file.txt" -Append
```

# NTP

## Non-regedit config

windows/directory/ntp.md

## Regedit config

### Set time service to use NTP

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters`

Type: `NTP`

### Enable NTP server

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer`

Enabled: 1

### Set Announce Flag

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config`

AnnounceFlags: 5

### Set external time source

`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters`

NtpServer: `dns.name.of.ntp.server,0x1` (space-separated list)

### Apply changes

Restart service, allow firewall:

```batch
net stop w32time
net start w32time

netsh advfirewall firewall add rule name="NTP-IN" dir=in action=allow protocol=UDP localport=123
```

## Microsoft guide

[https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/configure-authoritative-time-server](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/configure-authoritative-time-server)
