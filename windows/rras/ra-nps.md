<!-- 
---
title: "RRAS Server - RAVPN (NPS / EAP-MSCHAPv2)"
author: "Gergő Téringer"
---
 -->
# RRAS Server - RAVPN (NPS / EAP-MSCHAPv2)

Remote access VPN server with RRAS Server and StrongSwan Client using username/password authentication (EAP-MSCHAPv2) against Active Directory via NPS, instead of user certificates.

## 1. Information

The tunnel will be established between a Debian and a Windows host. Both share the network `200.0.0.0/24`, and the Windows Server also has their own network.

- Debian shared: `200.0.0.2`
- Windows shared: `200.0.0.1`
- Windows private: `10.0.0.103/24`

**Difference from the certificate guide:** IKEv2 on RRAS still requires a server certificate — this is a hard Windows requirement, PSK does not remove it. What changes here is the *client/user* authentication leg: instead of issuing and exporting a per-user certificate, users authenticate with their normal AD username and password using EAP-MSCHAPv2. No certificate template, PFX export, or `scp` of user credentials is needed on this path.

## 2. Server setup

### 2.1 Directory

In **Active Directory Users and Computers**, create a user **Security Group** called *VPN Users*. This group will hold the users you want to have remote access. Add a user to the group. No certificate enrollment permissions are needed on this group — it's used purely for NPS network-policy membership checks.

### 2.2 Certificate setup (server certificate only)

Only **one** certificate template is needed here — for the **server**. There is no *VPN User* template step, and no PFX export/distribution to the client.

Open **Certification Authority**, click on **Certificate Templates** > **Manage**. Copy a server certificate (e.g. Web Server), name it *VPN Server*, and make sure it has **Server Authentication** and **IPSec IKE intermediate** EKUs (Application policies).

Create a certificate for the VPN server. For the subject name, the guide will use `C=DK, O=LEGO, CN=srv-win.test.dk`. For alternate names:

- `DNS:srv-win.test.dk`
- `DNS:200.0.0.1`
- `IP4:200.0.0.1`

After creating the request, submit it, then install the certificate in the **Local Machine Personal** store.

> [!NOTE]
> The Windows IKEv2 implementation checks for a certificate with Server Authentication (and ideally IKE Intermediate) EKU in the local machine store when it starts the IKEv2 listener. Without one, IKEv2 will not come up — regardless of any preshared key or MS-CHAPv2 settings.

Copy the CA root certificate to the Debian client with `scp` — the client needs this to validate the server's certificate during IKE_AUTH.

### 2.3 Network Policy Server

Install the **Network Policy and Access Services** role to the server. After installation, open **Network Policy Server** console. Right click on **NPS (Local)**, and select *Register server in Active Directory*.

In the **Getting Started** section, select **Configure VPN or Dial-Up**. In the wizard:

- Select **Virtual Private Network (VPN) Connections**, then click **Next**.
- For clients, add localhost with IP address `127.0.0.1` (or any remote client).
- In **Authentication Methods**, uncheck *MS-CHAPv2* and select *EAP*. Click **Add**, then choose **Microsoft: Secured password (EAP-MSCHAP v2)** — do **not** choose "Microsoft: Smart Card or other certificate" (that path requires user certificates, which this guide avoids). There is no **Configure** step for this EAP type — no certificate needs to be selected here, since the server certificate for the tunnel is already handled by RRAS/IKEv2 itself in 2.2.
- Select the **VPN Users** group.
- For **IP Filters, Encryption Settings** and **Realm Name**, leave everything as **default**, then finish the wizard.

> [!NOTE]
> A common failure mode is leaving the wizard's default EAP type (Smart Card or other certificate) selected instead of switching it to Secured password (EAP-MSCHAP v2). If clients fail auth with "the authentication method used by the server to verify your username and password may not match the authentication method configured in your connection profile," this is the first thing to check.

### 2.4 Remote Access Server

Install the **Remote Access** role with **DirectAccess and VPN (RAS)** feature. In the configuration window, select **Deploy VPN only**.

Open the **Routing and Remote Access** management console, and select **Configure Routing and Remote Access**. Select **Custom configuration**, then check at least **VPN access**.

After setup, right click on **Ports** and select **Properties**. In **SSTP, L2TP, PPTP** and **PPPoE**, uncheck everything, so only IKEv2 and GRE are left.

By right-clicking on **Servername (local)**, open **Properties**:

- In **Security** tab, open **Authentication methods** and make sure only **EAP** and **IKEv2** are selected. Leave **Allow custom IPsec policy for L2TP/IKEv2 connection** *unchecked* — a PSK here is not required for IKEv2 to work, and combining it with EAP for the user leg is not a supported/tested Microsoft configuration.
- In **IPv4**, select **Static address** pool, and create a pool, for example `192.0.2.1-254`.

Your server is now ready to accept connections.

<!-- Created by: Gergő Téringer, 2026 -->