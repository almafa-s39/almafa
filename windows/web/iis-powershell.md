<!--
---
title: "IIS PowerShell Management"
author: "Simon Tamás"
---
-->

# IIS PowerShell Management

## 0. Information

There are two kinds of PowerShell modules for managing IIS. The old one is `WebAdministration`, the new one is `IISAdministration`. Both work. The guide will mix them as some tasks are easier with the old, others with the new syntax.

## 1. Installation

To manage IIS, you need to first install the IIS with management tools.

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```

## 2. Getting site info

To get website and binding information, use the following commands:

```powershell
Get-Website
Get-WebBinding -Name 'Default Web Site'
```

## 3. Configuring a new site

First, add a new site:

```powershell
New-Website -Name "NewSiteName" `
    -PhysicalPath "C:\root" `
    -ApplicationPool "DefaultAppPool" `
    -HostHeader 'www.example.com' `
    -Port 80
```

You can add/remove bindings with:

```powershell
New-IISSiteBinding -Name 'Default Web Site' `
  -BindingInformation '*:443:www.example.com' `
  -Protocol https `
  -CertificateThumbPrint $cert.Thumbprint `
  -CertStoreLocation 'Cert:\LocalMachine\My' `
  -SslFlag Sni
Remove-IISSiteBinding -Name 'Default Web Site' -BindingInformation '*:80:' -Protocol http
```

Explanation:

- Binding information follows the syntax: `<IP ADDRESS>:<PORT>:<HOST HEADER>`. The IP Address can be an asterisk (`*`) to bind on all IPs, and the host header field can be left empty to match for any host header.
- To bind a certificate to a binding, get the thumbprint of the certificate. You can do so by searching `Get-ChildItem cert:\LocalMachine\My`.
- The `-SslFlag Sni` option tells the binding to match for TLS SNI. This needs to be used if there are multiple HTTPS bindings on one server.

You can also edit the file system location of the website, or one of its virtual directories/subdirectories:

```powershell
Set-ItemProperty 'IIS:\Sites\Default Web Site' -Name physicalPath -Value 'D:\www\example'
Set-ItemProperty 'IIS:\Sites\Default Web Site\api' -Name physicalPath -Value 'D:\www\api'
```

<!-- Created by: Simon Tamás, 2026 -->
