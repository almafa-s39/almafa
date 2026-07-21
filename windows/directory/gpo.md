<!-- 
---
title: "Group Policy Objects (GPO) Reference"
author: "Gergő Téringer"
---
-->
# Group Policy Objects (GPO) Reference

Group Policy Objects (GPOs) allow administrators to centrally manage and enforce configurations across a Windows Active Directory environment. This reference guide is divided into Computer Configuration (applied to machine objects regardless of who logs in) and User Configuration (applied to user objects regardless of which machine they use).

## 1. Computer Settings

These policies must be linked to an Organizational Unit (OU) containing Computer objects. They are typically applied during system startup or during the background refresh cycle.

### 1.1 Password Policies

`Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Password Policy`

> [!NOTE]  
> If you want to set the minimum password length to more than 14 characters on modern Windows operating systems, you must first enable the **Relax minimum password length limits** policy in the same directory. Otherwise, the traditional length policy caps out at 14.

### 1.2 Disable Local Administrator

There are two ways to handle the built-in local Administrator account. Using Group Policy Preferences (GPP) allows you to update the account status, while Security Options forcefully disables it across the board.

**Method A: Group Policy Preferences (Recommended for precise control)**
`Computer Configuration > Preferences > Control Panel Settings > Local Users and Groups`

> [!NOTE]  
> Add a new local user configuration with the following settings to disable the built-in account:
>
> - **Action:** Update
> - **User name:** Administrator
> - **Account is disabled:** Checked

**Method B: Security Options**
`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`

> [!NOTE]  
> Set **Accounts: Administrator account status** to `Disabled`.

### 1.3 Disable CTRL+ALT+DEL Requirement

`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`

> [!NOTE]  
> Set **Interactive logon: Do not require CTRL+ALT+DEL** to `Enabled`. This is often used for virtualized environments or kiosk machines to streamline the login process.

### 1.4 Interactive Logon Banner

`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`

> [!NOTE]  
> Configure both the **Message title** and **Message text** for users attempting to log on. This displays a mandatory legal or informational prompt that users must accept before they can enter their credentials.

### 1.5 Prevent LM Hash Storage

`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options`

> [!NOTE]  
> Set **Network security: Do not store LAN Manager hash value on next password change** to `Enabled`. LAN Manager (LM) hashes are extremely weak and easily cracked. This prevents Windows from storing legacy hashes in the SAM Database and Active Directory.

### 1.6 Allow Users to Log in to a Domain Controller

`Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment`

> [!NOTE]  
> Open **Allow log on locally** and add the specific Users or Security Groups you want to grant interactive login permissions to. By default, standard users cannot log into a Domain Controller desktop.

### 1.7 Set Environment Variables

`Computer Configuration > Preferences > Windows Settings > Environment Variables`

> [!NOTE]  
> Click Add and define the System variables you need (e.g., custom Java paths or application directories). Variables set here apply globally to the system, whereas setting them in the User Configuration only applies them to the specific user's profile.

### 1.8 Disable First Sign-in Animation

`Computer Configuration > Policies > Administrative Templates > System > Logon`

> [!NOTE]  
> Set **Show first sign-in animation** to `Disabled`. This skips the "Hi, we're getting things ready for you" screen on Windows 10/11, significantly speeding up the initial profile generation time.

### 1.9 Turn Off File History

`Computer Configuration > Policies > Administrative Templates > Windows Components > File History`

> [!NOTE]  
> Set **Turn off File History** to `Enabled`. This prevents users from utilizing local disk space or network drives for native Windows file versioning, which is useful when you rely on centralized backup solutions instead.

### 1.10 Windows Update Configuration

`Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Update`

> [!NOTE]  
> This directory contains all WSUS and Update Ring configurations. Configure your intranet update service location, active hours, and automatic download/install schedules here.

### 1.11 Automatic Mapping (Home Directory)

Setting up a dynamic home directory requires a combination of strict NTFS file server permissions, a Computer Policy to define the path, and a User Preference to map it cleanly.

**1. File Server NTFS Permissions for the Root Folder:**

- Disable inheritance on the root folder.
- **Remove** everything except `CREATOR OWNER` and `SYSTEM`.
- **Add** `Domain Users`.
- Edit `Domain Users`, click **Show advanced permissions**, and set:
  - **Applies to:** `This folder only`
  - **Permissions:** Check `Create folders / Append data`, `Traverse folder / execute file`, and `List folder / read data`.

**2. File Server Share Permissions:**

- Grant `Full Control` to `Authenticated Users`. (Security will be restricted by the NTFS permissions above).

**3. Configure the Home Folder Path (Computer Policy):**
`Computer Configuration > Policies > Administrative Templates > System > User Profiles`

> [!NOTE]  
> Set **Set user home folder** to `Enabled`.
>
> - **Path:** `\\fqdn\SHARENAME`
> - **Drive letter:** Choose your preferred letter.

**4. Rename the Mapped Drive (User Preference):**
`User Configuration > Preferences > Windows Settings > Drive Maps`

> [!NOTE]  
> The native GPO maps the drive with a messy network path name. Use this GPP to rename it cleanly for the user:
>
> - **Action:** Replace
> - **Location:** `\\SHARE-COMPUTER\SHARENAME\%username%.%userdomain%`
> - **Reconnect:** Checked
> - **Label as:** Whatever you prefer (e.g., `Personal Space (%username%)`)
> - **Drive Letter:** Set to the same letter used in the Computer Policy.

### 1.12 Kerberos Hardening

`Computer Configuration > Policies > Administrative Templates > System > Kerberos`

> [!NOTE]  
> Set **Kerberos client support for claims, compound authentication and Kerberos armoring** to `Enabled`. This is required if you are utilizing Dynamic Access Control (DAC) or need enhanced protection against pass-the-hash attacks.

### 1.13 Remote Desktop Session Limits

`Computer Configuration > Policies > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Connections`

> [!NOTE]  
> Set **Restrict Remote Desktop Services users to a single Remote Desktop Services session** to `Disabled`.
>
> You can also configure the **Limit number of connections** attribute in the same directory. However, be aware that without the **RD Session Host** role installed and proper RDS CALs (licenses) activated, Windows Server natively hard-caps concurrent administrative RDP sessions to a maximum of two, regardless of this GPO setting.

---

## 2. User Settings

These policies must be linked to an Organizational Unit (OU) containing User objects. They apply dynamically when the user logs into any domain-joined machine.

### 2.1 Automatic Program Start

`User Configuration > Policies > Windows Settings > Scripts (Logon/Logoff) > Logon`

> [!NOTE]  
> Click Add to define a new startup script or executable.
>
> - **Script name:** Define the full path to the executable or `.ps1`/`.bat` file.
> - **Script parameters:** Define any required launch arguments.

### 2.2 Force Desktop Wallpaper

`User Configuration > Policies > Administrative Templates > Desktop > Desktop`

> [!NOTE]  
> Open **Desktop Wallpaper** and set the path to an image file. The image must be hosted on a highly available network share (e.g., `\\domain.local\NETLOGON\wallpaper.jpg`) that all users have `Read` access to.

### 2.3 Disable Registry Editor

`User Configuration > Policies > Administrative Templates > System`

> [!NOTE]  
>
> - Set **Prevent access to registry editing tools** to `Enabled`. This blocks `regedit.exe`.
> - For additional lockdown, you can open **Don't run specified Windows applications** and explicitly add `regedit.exe` to the block list.

### 2.4 Disable CMD, Run, and PowerShell

`User Configuration > Policies > Administrative Templates > Start Menu and Taskbar`

> [!NOTE]  
> Set **Remove Run menu from Start Menu** to `Enabled`.

`User Configuration > Policies > Administrative Templates > System`

> [!NOTE]  
>
> - Set **Prevent access to the command prompt** to `Enabled`. (This natively blocks `cmd.exe` and batch scripts).
> - Open **Don't run specified Windows applications**, set it to `Enabled`, and add `powershell.exe`, `powershell_ise.exe`, and `pwsh.exe` to strictly block PowerShell access.

### 2.5 Disable Portable Media Drives

`User Configuration > Preferences > Control Panel Settings > Devices`

> [!WARNING]  
> The specific hardware class you want to disable must be physically connected to the machine (or Domain Controller) you are using to author this GPO, as the wizard pulls from locally installed device drivers to populate the selection menu.
> [!NOTE]  
> Add a new device restriction with the following configuration:
>
> - **Action:** Update
> - **Device class:** Select `CD/DVD Drive`, `Disk Drives` (USB), or `Floppy`.
> - Check **Disable**.
> You can only select one device class per entry. Press Apply, then add a new item for each additional class you want to block.

### 2.6 Create Shortcuts for Users

`User Configuration > Preferences > Windows Settings > Shortcuts`

> [!NOTE]  
> Add a new shortcut to push standardized icons to the user's Desktop or Start Menu:
>
> - **Name:** The display name of the shortcut.
> - **Target type:** `File System Object` (for apps) or `URL` (for web links).
> - **Location:** Where the shortcut should appear (e.g., `Desktop`).
> - **Target Path:** The exact path to the executable (e.g., `C:\path\to\your\exe`).
> You can also define custom icon files (.ico) within this preference.

### 2.7 Standard Network Drive Mapping

`User Configuration > Preferences > Windows Settings > Drive Maps`

> [!NOTE]  
> Use this section to map standard departmental shares (e.g., an `S:\` drive for Sales).
>
> - **Action:** Update or Replace.
> - Provide the UNC path to the share.
> - Check **Reconnect**.
> - Assign a specific drive letter and a friendly label.

<!-- Created by: Gergő Téringer, 2026 -->