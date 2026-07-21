<!-- 
---
title: "MSFT Connect Test"
author: "Gergő Téringer"
---
-->
# MSFT Connect Test

The Network Connectivity Status Indicator (NCSI) is a Windows component that determines whether a computer has internet or intranet connectivity. In isolated or air-gapped environments, Windows may falsely report "No Internet Access" (often displaying a globe icon on the taskbar), which can cause native services like Microsoft Office, Windows Update, and Outlook to time out or malfunction.

By hosting these specific DNS zones and web files internally, you effectively spoof the Microsoft validation servers. This tricks the Windows clients into registering a successful internet connection, eliminating the false warnings and optimizing background service timeouts.

## 1. DNS Zones and Records

Windows performs an active DNS probe to verify connectivity before attempting an HTTP request. You must create the authoritative zones that Windows queries and point the specific A records to your internal web server.

**Configuration via PowerShell:**

```powershell
# Create the authoritative DNS zones
Add-DnsServerForwardZone -Name "msftconnecttest.com" -ZoneFile "msftconnecttest.com.dns"
Add-DnsServerForwardZone -Name "msftncsi.com" -ZoneFile "msftncsi.com.dns"

# Create the A records pointing to your internal web server (assuming IP 10.10.10.50)
Add-DnsServerResourceRecordA -ZoneName "msftconnecttest.com" -Name "www" -IPv4Address "10.10.10.50"
Add-DnsServerResourceRecordA -ZoneName "msftncsi.com" -Name "dns" -IPv4Address "10.10.10.50"
```

**Command Breakdown & Explanation:**

- `msftconnecttest.com`: The primary zone used by modern Windows 10/11 and Windows Server 2016+ clients to test TCP/HTTP connectivity.
- `msftncsi.com`: The legacy zone used by Windows 7, Windows 8, and older server operating systems for active DNS probing.
- `www.msftconnecttest.com` and `dns.msftncsi.com`: The specific hosts Windows looks for. Directing these A records to your internal IIS server ensures the subsequent HTTP requests land in the right place. (Active DNS probe)

## 2. Web Server Setup

Once DNS resolves, Windows attempts to download a specific text file. The HTTP response must be exactly what Microsoft's public servers would return, otherwise the NCSI process fails.

**Configuration via PowerShell:**

```powershell
# Install the native IIS Web Server role
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Create the text file for Windows 10 1607 and newer
Set-Content -Path "C:\inetpub\wwwroot\connecttest.txt" -Value "Microsoft Connect Test" -NoNewline

# Create the text file for older Windows versions
Set-Content -Path "C:\inetpub\wwwroot\ncsi.txt" -Value "Microsoft NCSI" -NoNewline
```

**Command Breakdown & Explanation:**

- `Install-WindowsFeature`: Installs the native IIS role to serve the files over port 80.
- `connecttest.txt`: Modern Windows clients request `http://www.msftconnecttest.com/connecttest.txt`. The file must contain the exact string `Microsoft Connect Test`.
- `ncsi.txt`: Legacy clients request `http://www.msftncsi.com/ncsi.txt`. The file must contain the exact string `Microsoft NCSI`. (Note: While modern systems check `connecttest.txt`, older legacy Windows explicitly looks for the `ncsi.txt` filename rather than reusing `connecttest.txt`).
- `-NoNewline`: This is a highly critical parameter. If the text file contains a carriage return or hidden newline character at the end, Windows will reject the payload and declare the internet disconnected.

<!-- Created by: Gergő Téringer, 2026 -->