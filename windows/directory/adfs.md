<!-- 
---
title: "AD FS (Authentication services)"
author: "Gergő Téringer"
---
-->
# AD FS (Authentication services)

## 1. Introduction

A few variables will be needed, they are the following:

- **ADFS server's hostname:** `server1.company.com`
- **ADFS service name:** `sso.company.com`

## 2. Certificates

In your enterprise CA, create a new certificate template by **duplicating the existing Web Server template** with the following changes:

- **Template name:** ADFS Certificate
- Check **Publish certificate in Active Directory**
- In the **Security** tab, add **Enroll** permission to **Authenticated Users**
- In the **Request Handling** tab, check **Allow private key to be exported**

Issue the new template with the **New > Certificate Template to Issue** wizard.

Open the Local Computer certificate store on the server where ADFS will be installed. In the **Personal** store, right-click **All Tasks > Request New Certificate**. Select the newly created template and click on **Properties**. Make the following changes:

- As the **Subject name**, enter `CN=server1.company.com` as the **Distinguished Name**.
- For the **Subject Alternative Name**, create the following fields (all of type **DNS**):
  - `sso.company.com`
  - `certauth.sso.company.com`
  - `server1.company.com`
  - `enterpriseregistration.company.com`
- In the **Private key** tab, check **Make private key exportable**
- Set a memorable friendly name

Enroll the certificate.

> [!NOTE]
> Because ADFS will use sso.company.com, this needs to be a **domain record pointing to the ADFS server**.

## 3. Installation

From **Server Manager**, add the **Active Directory Federation Services** role.

After the role is installed, open the configuration wizard and use the following settings:

- Select **Create the first federation server in a federation server farm**
- Make sure a domain admin user is selected
- Select the SSL certificate from the dropdown
- The **Federation Service Name** should be selected from the second dropdown, in this case, `sso.company.com`
- Enter a display name, e.g. **Company SSO**
- Before navigating to the next page, issue the following **PowerShell** command:

```powershell
Add-KdsRootKey -EffectiveTime (Get-Date).AddHours(-10)
```

- Now select **Create a Group Managed Service Account**, and enter a name, e.g. **adfssvc**
- Leave all other options as default and finish the wizard

ADFS Setup is now complete.

## 4. Troubleshooting

If the `Add-KdsRootKey` command fails, check if the DC is really in the default **Domain Controllers** organizational unit. Also, try to issue the same command from another domain-joined server with domain admin privileges. *(The PShell cmdlets might need RSAT-ADDS to be installed)*

If the service fails to start due to a logon failure with the service account, you have the following options:

- For a **quick fix**, open **Services**, ADFS, "Log On" tab, clear the password and hit Ok.
- For a **permanent fix**, issue the following command:

```powershell
sc.exe managedaccount adfssrv true
```

<!-- Created by: Gergő Téringer, 2026 -->