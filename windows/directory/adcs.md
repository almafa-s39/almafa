<!-- 
---
title: "Two-tiered ADCS AIA-CDP"
author: "Gergő Téringer"
---
-->
# Two-tiered ADCS AIA-CDP

## 1. Information

This guide will demonstrate how to create a two-tiered Windows AD CS (Active Directory Certificate Services) PKI with CDP (CRL Distribution Point) and AIA (Authority Information Access) extensions set up correctly.

A two-tier architecture is a security best practice: the Root CA is kept offline to protect the core cryptographic identity of the domain, while the Subordinate (Issuing) CA remains online to handle day-to-day certificate requests.

Two Windows Server 2022 VMs will be used:

- srv-root - 10.0.0.1/24 (Offline Root CA)
- srv-signing - 10.0.0.2/24 (Online Enterprise Issuing CA)

## 2. Prerequisites

As with any PKI setup, always make sure that the current time is correct on all machines.

For the enterprise issuing CA, you will need **AD DS** installed, in this case, on **srv-signing**. This guide will use the domain **lego.dk**. The server was promoted to be the **DC** of this forest.

For name resolution, **srv-signing** is set as the DNS server for **srv-root**.

This setup will use the **FQDN** `pki.lego.dk` for distributing certificates. Create an *A/CNAME* record for it that points to **srv-signing**.

## 3. Setup

### 3.1 IIS

First, we need to set up a **web server** to serve as a distribution point. On **srv-signing**, **install IIS with default settings**.

**Create** the folder `C:\pki`. **Right click** on it, and click on ***Give access to people...* > *Specific People***. **Add** the following members:

- **Cert Publishers** - Read/Write
- **ANONYMOUS LOGON** - Read
- **Everyone** - Read

**Open** the IIS management console. Under the **default website**, create a new **virtual directory** with the following parameters:

- **Alias:** `pki`
- **Physical path:** `C:\pki`

In the virtual directory menu, double click on **Request Filtering**. On the right side, **edit feature settings**. In the pop-up window, make sure **Allow double escaping** is checked.

In the IIS tree view on the left, click **SRV-SIGNING** *(your server name)* and on the right, click **Restart**.

IIS is now ready to serve requests.

### 3.2 Root certification authority

On **srv-root**, install **AD CS** with all default settings. Only install the **Certification Authority** role feature.

When setting up the authority, select **Standalone CA** *(you cannot select Enterprise CA as this machine is not domain-joined)* and **Root CA** options.

When entering the **name** of the CA, for convenience, **DO NOT use any whitespace**. The guide will use `lego-rca`.

After the setup is complete, open the **Certification Authority** management console, then right-click on the CA and select **Properties**. Open the **Extensions** pane.

**With the CDP extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(For the variable part, just follow the example)*
  - With this location selected, **check**:
    - *Include in CRLs. Clients use this to find Delta CRL locations.*
    - *Include in the CDP extension of issued certificates.*

> [!WARNING]
> **CDP** and **AIA** extensions only work over **HTTP** and **not HTTPS**!
> If you enforce HTTPS for a CRL download, the client must first verify the SSL certificate of the web server. To verify that SSL certificate, it must download a CRL, which requires an HTTPS connection, creating an impossible circular dependency.

**With the AIA extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/lego-rca<CertificateName>.crt`
  - With this location selected, **check**:
    - *Include in the AIA extension of issued certificates.*

Click **OK** and **restart** the CA when prompted.

In a terminal with admin privileges, issue the following command to manually generate the first CRL:

```ps1
certutil -crl
```

Navigate to `C:\Windows\System32\Certsrv\CertEnroll\` in File Explorer. There should be two files, with **.crt** and **.crl** file extensions. Copy both files to the `pki` share on **srv-signing**. Rename the root certificate file to `lego-rca.crt`.

> [!NOTE]
> The root certificate filename should match the filename you wrote in the AIA extension.
>
> In that location, the `<CertificateName>` variable only contains text after the root CA is renewed, right now it is an empty string.

## Issuing certification authority

After the root certification authority is set up, you can move on to configuring the issuing (subordinate) certification authority. The first task is to **import the root certificate** to the **Trusted Root Certification Authorities** store. The file is already present at `C:\pki\lego-rca.crt`. Right click and install it. *(Local Machine, Trusted Root Certification Authorities)*

> [!IMPORTANT]
> The root certificate needs to be trusted otherwise importing the signed subordinate certification authority certificate will fail.

Now **install AD CS** role on the server, only selecting the **Certification Authority** role feature.

**When configuring the role, select:**

- Enterprise CA
- Subordinate CA
- For convenience, select a name **WITHOUT whitespace characters**. This guide will use `lego-signing-ca`

**Export the signing request** into a file. Copy that file over to **srv-root** and in its CA console, **submit the request**. After submission, **issue** the certificate from the pending certificates. After issuing, **open the certificate** *(in Issued Certificates)* and **export it** into a file. *(Details pane > Copy to File...)*

**Copy** the signed certificate back to **srv-signing**, then in the CA console, **install the certificate**. If everything so far was done correctly, there should be no warnings given, because the certificate engine should be able to reach the CDP of the root cert to check the validity of the subordinate certificate.

After the Certification Authority service has started, right click on the CA and open **Properties**. Open the **Extensions** pane.

**With the CDP extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(For the variable part, just follow the example)*
  - With this location selected, **check**:
    - *Include in CRLs. Clients use this to find Delta CRL locations.*
    - *Include in the CDP extension of issued certificates.*
- **Add a new location:** `C:\pki\<CaName><CRLNameSuffix><DeltaCRLAllowed>.crl` *(The filename part is the same)*
  - With this location selected, **check**:
    - *Publish CRLs to this location.*
    - *Publish Delta CRLs to this location.*

> [!TIP]
> To publish the CRL to a **remote location**, you can use the `file:\\server-name\share-name\...` format after creating the share and granting write access for the **Cert Publishers** group.

**With the AIA extension selected:**

- **Delete** all locations **except the first one** *(starting with `C:\...`)*
- **Add a new location:** `http://pki.lego.dk/pki/lego-signing-ca<CertificateName>.crt`
  - With this location selected, **check**:
    - *Include in the AIA extension of issued certificates.*

Click **OK** and **restart** the CA when prompted.

In a terminal with admin privileges, issue the following command:

```ps1
certutil -crl
```

Navigate to `C:\Windows\System32\Certsrv\CertEnroll\` in File Explorer. There should be two files, with **.crt** and **.crl** file extensions. Copy both files to `C:\pki\`. *(Although the .crl file should already be present, after running the certutil command.)* **Rename** the certificate file to `lego-signing-ca.crt`.

Now, you have **CDP** and **AIA** extensions configured, clients will be able to check revocation and obtain the certificate chain to verify trust.

## 4. Verification

To verify your setup, open `pkiview.msc` (Enterprise PKI tool) on srv-signing. After loading, all of the **Status** fields should read **OK** when navigating through the trust chain. This confirms that the CA certificates, AIA locations, and CDP locations are all accessible and properly formatted.

## 5. Troubleshooting

If `pkiview.msc` displays errors, or if clients fail to trust issued certificates, the issue is almost always related to the HTTP distribution point.

### 5.1 Verifying IIS Reachability

Ensure that the web server is actually serving the files. Open a web browser on any domain-joined client and navigate directly to the URLs configured in your extensions:

- `http://pki.lego.dk/pki/lego-rca.crt`
- `http://pki.lego.dk/pki/lego-rca.crl`

If the browser returns a **404 Not Found**, verify the physical files are in C:\pki and exactly match the filenames requested. If it returns a **403 Forbidden or 404.11**, double-check your IIS double-escaping settings.

### 5.2 Deep Certificate Verification using Certutil

You can use the built-in Windows certificate utility to diagnose exactly why a certificate is failing the trust chain or revocation check. Run this on a client machine using an issued certificate.

**Command:** `certutil -verify -urlfetch "C:\path\to\issued_certificate.crt"`

**What it checks and variables to look for:**

- The output will step through every link in the chain (from the client cert, to the issuing CA, to the root CA).
- It tests the AIA and CDP URLs explicitly. Look for the "----------------  Certificate AIA  ----------------" and "----------------  Certificate CDP  ----------------" sections.
- If it says "Failed 'The server name or address could not be resolved'", you have a DNS issue with pki.lego.dk.
- If it says 'Verified', the extension is working perfectly.

### 5.3 Forcing a Local CRL Cache Clear

Windows aggressively caches CRLs. If you fixed an IIS issue or published a new CRL but clients are still throwing revocation errors, you must clear the local client cache to force them to download the new files.

**Command:** `certutil -urlcache * delete`

**What it checks and variables to look for:**

- This command flushes the local URL cache for the current user/system. Afterward, any new certificate validation attempts will be forced to reach out to the HTTP CDP and fetch the latest list.

<!-- Created by: Gergő Téringer, 2026 -->