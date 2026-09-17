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
  - `TEST-SRV-1.test.com` (DC, Root CA)
  - `TEST-SRV-2.test.com` (Sub CA)
- CA:
  - `ROOT-CA` (Standalone Root CA)
  - `SUB-CA` (Enterprise Subordinate CA)

The root certification authority is already set up, this guide will only cover setting up the subordinate CA.

## 1. Installation

Install AD CS on the subordinate CA:

```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools
```

Make sure the Root CA certificate is trusted by the subordinate CA. You can achieve this by using GPOs or copying the root certificate to the machine and issuing the following command:

```powershell
certutil -addstore -f Root C:\certs\RootCA.crt
```

Now, set up the subordinate CA. You can also give the parameters inline to the command.

The `ParentCA` option is only necessary if the parent CA is an online enterprise CA, in which case the signing will be done automatically. You can get the exact value for this by issuing `certutil -dump` on the root ca machine and looking for the value of `Config`.

```powershell
$params = @{
    CAType              = 'EnterpriseSubordinateCA'
    CACommonName        = 'SUB-CA'
    CADistinguishedNameSuffix = 'dc=test,dc=com'
    CryptoProviderName  = 'RSA#Microsoft Software Key Storage Provider'
    KeyLength           = 4096
    HashAlgorithmName   = 'SHA256'
    ParentCA            = 'TEST-SRV-1.test.com\ROOT-CA'
    DatabaseDirectory   = 'C:\Windows\System32\CertLog'
    LogDirectory        = 'C:\Windows\System32\CertLog'
    Force               = $true
}
Install-AdcsCertificationAuthority @params
```

If the root CA is an offline/standalone CA, this command will output a request file. Copy that file the root ca machine, then issue the following commands there:

```powershell
certreq -submit -config "TEST-SRV-1.test.com\ROOT-CA" C:\certs\SubCA.req
# Note the RequestId returned, then approve and retrieve:
certutil -resubmit <RequestId>
certreq -retrieve -config "TEST-SRV-1.test.com\ROOT-CA" <RequestId> C:\certs\SubCA.crt
```

Copy the signed certificate back to the subordinate CA, then install it for the service:

```powershell
certutil -installcert C:\certs\SubCA.crt
Start-Service certsvc
```

## 2. Configuring the Subordinate CA

### 2.1 Setting CDP/AIA Values

You can set CDP/AIA values for the CA with the following commands:

```powershell
# Inspect current values first
Get-CACrlDistributionPoint | Format-List
Get-CAAuthorityInformationAccess | Format-List

# Remove all existing entries
Get-CACrlDistributionPoint | Remove-CACrlDistributionPoint -Force
Get-CAAuthorityInformationAccess | Remove-CAAuthorityInformationAccess -Force

# Local publish location
Add-CACrlDistributionPoint -Uri 'C:\Windows\System32\CertSrv\CertEnroll\%3%8%9.crl' `
    -PublishToServer -PublishDeltaToServer -Force

# HTTP (adjust to your published PKI web path)
Add-CACrlDistributionPoint -Uri 'http://pki.contoso.com/pki/%3%8%9.crl' `
    -AddToCertificateCdp -AddToFreshestCrl -Force

Add-CAAuthorityInformationAccess -AddToCertificateAia `
    -Uri 'http://pki.contoso.com/pki/%1_%3%4.crt' -Force
```

The following variables were used for the URIs:

- `%1`: `<ServerDNSName>`
- `%3`: `<CAName>`
- `%4`: `<CertificateName>`
- `%8`: `<CRLNameSuffix>`
- `%9`: `<DeltaCRLAllowed>`

### 2.2 Setting CA periods

You can set CA time periods with the following commands. The following values are the default values.

```powershell
certutil -setreg CA\ValidityPeriod "Years"
certutil -setreg CA\ValidityPeriodUnits 2 # This is the max age a signed cert can get by this CA
certutil -setreg CA\CRLPeriod "Weeks"
certutil -setreg CA\CRLPeriodUnits 1
certutil -setreg CA\CRLDeltaPeriod "Days"
certutil -setreg CA\CRLDeltaPeriodUnits 1

# Restart the service to apply changes
Restart-Service certsvc
# Issue a CRL
certutil -crl
```

<!-- Created by: Simon Tamás, 2026 -->
