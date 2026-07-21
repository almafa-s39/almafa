<!-- 
---
title: "DNSSEC"
author: "Gergő Téringer"
---
-->
# DNSSEC

Domain Name System Security Extensions (DNSSEC) adds cryptographic signatures to existing DNS records. These signatures ensure that the DNS information hasn't been tampered with and originated from the authorized domain owner. Implementing DNSSEC properly prevents cache poisoning and man-in-the-middle attacks. This guide outlines the setup for a multi-tier DNS infrastructure, ensuring secure name resolution compatible with modern Windows Server 2022 and Server 2025 environments.

## 1. Zone Creation and Signing

First, create the top-level and root zones on your internet-facing DNS server (INET), and then digitally sign all applicable zones across both your INET and Domain Controller (DC) servers.

You must create the following primary forward lookup zones on the INET server:

- `.` (root)
- `dk`
- `net`

You can sign these zones using the native DNS Manager GUI (DNSSEC > Sign zone wizard) utilizing the default settings, or automate the process via PowerShell.

```powershell
Invoke-DnsServerZoneSign -ZoneName "skillsdev.dk" -SignWithDefault
```

**Command Breakdown & Explanation:**

- `Invoke-DnsServerZoneSign`: Initiates the cryptographic signing of a specific zone using the built-in Windows Key Master.
- `-SignWithDefault`: Bypasses custom parameter prompts and uses standard key lengths and algorithms suitable for modern Active Directory and Windows Server environments.

## 2. Delegating Trust (DS Records)

After a zone is signed, you must establish a chain of trust. This is done by taking the Delegation Signer (DS) record of the child zone and importing it into its respective parent zone.

When Windows signs a zone, it automatically generates a text file containing the DS records. The file is saved with the following naming convention: `C:\Windows\System32\dns\dsset-<zone_name>`

> [!NOTE]
> For the root zone (`.`), the zone_name variable is an empty string, meaning the file will simply be named `dsset-`.

You must import each file into the corresponding parent zone. Depending on your architecture, you may need to securely copy (e.g., using SCP) these files between servers before importing.

```powershell
# This command imports the DS records for the 'dk' zone into the root '.' zone
Import-DnsServerResourceRecordDS `
  -ZoneName "." `
  -DSSetFile "C:\Windows\System32\dns\dsset-dk"
```

**Command Breakdown & Explanation:**

- `Import-DnsServerResourceRecordDS`: Reads the cryptographic hash from the specified file and injects it into the parent zone.
- `-ZoneName`: Specifies the target parent zone that will host the DS record (in this case, the root zone `"."`).
- `-DSSetFile`: Specifies the absolute path to the generated DS set file belonging to the child zone.

## 3. Importing Trust Anchors

After the entire chain is created, you must import the keyset of the root zone into any DNS resolver where you want the DNSSEC chain to be explicitly trusted (for example, your INET, DC, and DEV-SRV machines).

This can be done using the "Import DNSKEY" wizard under the Trust Points section in the DNS Manager, or via PowerShell.

```powershell
Import-DnsServerTrustAnchor -KeySetFile "C:\Windows\System32\dns\keyset-"
```

**Command Breakdown & Explanation:**

- `Import-DnsServerTrustAnchor`: Injects the public cryptographic key (DNSKEY) of the root zone into the local server's Trust Points, validating the entire downward chain.
- `-KeySetFile`: Points to the root zone's keyset file automatically generated during the initial signing process.

## 4. Troubleshooting and Verification

To test if DNSSEC is working correctly and the trust chain is valid, you can force a DNS query to explicitly request the security extensions.

```powershell
Resolve-DnsName -Name www.skillspublic.dk -DnsSecOk
```

**What it checks and variables to look for:**

- `Record Output`: A successful query will return the standard record value (e.g., an `A` record IP) along with the `RRSIG` (Resource Record Signature) data.
- `Query Status`: If the signature is invalid, expired, or the trust chain is broken anywhere up to the root, the command will return a `SERVFAIL` exception. This explicitly means the cryptographic verification failed, not necessarily that the target host is offline.

<!-- Created by: Gergő Téringer, 2026 -->