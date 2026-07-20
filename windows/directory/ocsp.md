# OCSP

The Online Certificate Status Protocol (OCSP) provides a more efficient, real-time alternative to Certificate Revocation Lists (CRLs). Instead of clients downloading an entire list of revoked certificates, they send a lightweight HTTP query to the OCSP responder for the status of a specific certificate. The responder replies with a cryptographically signed "Good", "Revoked", or "Unknown" status.

## 1. Installation

**Install** the OCSP responder feature. To do this, install the **Online Responder** feature of the **Active Directory Certificate Services** role from *Server Manager*, or enter the following command:

```powershell
Install-WindowsFeature ADCS-Online-Cert -IncludeManagementTools
```

After the installation is complete, open and complete the configuration wizard. No configuration is needed.

## 2. Certificate

When the client asks the server about certificate status, it receives a signed response. For this, the OCSP server needs an ***OCSP Response Signing*** certificate.

In the **Certification Authority** console, right-click on **Certificate Templates** and select **Manage**. Duplicate the existing *OCSP Response Signing* template and change the following settings:

- Set a name, e.g. `ocsp_v2`
- In **Request Handling** tab:
  - Check **Allow private key to be exported**
  - Click on **Key Permissions**, and add the **NETWORK SERVICE** account with full permissions
- In **Security** tab, add the computer which the CA is running on and give it **Read, Enroll** rights

Now in **Certification Authority**, right-click on **Certificate Templates** and select **New > Certificate Template to Issue**. Select and issue the newly created template.

Now the OCSP URL needs to be set in the Certification Authority, so issued certificates will contain the OCSP field. To do this, right-click on your authority and select **Properties**. Select the **Extensions** tab, then the **Authority Information Access (AIA)** item from the dropdown list. Add a new entry, with the value `http://ocsp.company.com/`. With the new entry selected, check **Include in the online certificate status protocol (OCSP) extension**. After closing, confirm restarting the service.

> [!NOTE]
> By default the OCSP responder is on the subpath **http:/<span>/ser</span>ver/ocsp** of the server, but later we will change it to a separate virtual host, in this case **http:/<span>/ocsp.company</span>.com/**.
>
> *This means you need to make sure there is a DNS record **ocsp<span>.company</span>.com** pointing to the CA.*

## 3. Online Responder Configuration

Open the **Online Responder Management** console, right-click on **Revocation Configuration** and select **Add Revocation Configuration**. Go through the wizard:

- Specify a **name**, e.g. *Company - OCSP*
- Select **Select a certificate for an Existing enterprise CA**
- Select **Browse CA certificates published in Active Directory**, then select your CA certificate in the browser
- Choose **Automatically select a signing certificate**
- Make sure *Auto-Enroll for an OCSP signing certificate* is **checked**
- Select the new certificate template
- Click on the **Provider** button and make sure your existing CDP is there

After finishing the wizard you should see a green checkmark meaning that OCSP is working.

## 4. Custom responder URL

To move the OCSP responder to a custom URL, in this case, `ocsp.company.com`, open **IIS Management** console and create a new site with the following settings:

- **Name:** ocsp
- **Application Pool:** OCSPISAPIAppPool
- **Physical path:** `C:\Windows\SystemData\ocsp`
- **HTTP binding:** ocsp<span>.company.</span>com

Create the site and **exit** the IIS console.

Open <kbd>C:\Windows\System32\inetsrv\applicationHost.config</kbd> in a text editor and towards the end, locate the following line:

```xml
<location path="Default Web Site/ocsp">
```

Change this to:

```xml
<location path="ocsp">
```

After restarting the web server, the OCSP responder should be responding at the new URL.

## 5. Testing

To test all configuration, open `pkiview.msc`. Everything should be OK.
