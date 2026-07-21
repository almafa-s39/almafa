<!-- 
---
title: "Applocker"
author: "Gergő Téringer"
---
-->
# Applocker

AppLocker is an application whitelisting technology that restricts which programs users can execute based on the path, publisher, or file hash.

Configuring AppLocker incorrectly is one of the easiest ways to completely brick a Windows installation. If you enable enforcement without defining baseline rules, Windows will block its own core components (including cmd.exe, explorer.exe, and the Start Menu), rendering the system unusable. Modern operating systems like Windows 11 and Server 2025 rely heavily on Universal Windows Platform (UWP) apps for core UI components, which requires specific care.

## 1. Enabling the Application Identity Service

AppLocker rules are completely ignored by the operating system unless the Application Identity (`appidsvc`) service is running. This service is responsible for verifying the attributes of an application against your configured policies.

**GPO Configuration Path:**
`Computer Configuration > Policies > Windows Settings > Security Settings > System Services`

**Configuration Steps:**

1. Locate Application Identity in the services list.
2. Double-click it and check "Define this policy setting".
3. Set the startup mode to Automatic.

## 2. Generating Safe Default Rules

Before creating any custom Deny rules, you must generate the Default Rules. These rules ensure that Windows system files and Administrator accounts are explicitly permitted to run executables.

**GPO Configuration Path:**
`Computer Configuration > Policies > Windows Settings > Security Settings > Application Control Policies > AppLocker`

**Configuration Steps:**

1. Expand the AppLocker node.
2. Right-click Executable Rules and select Create Default Rules.
3. Right-click Windows Installer Rules and select Create Default Rules.
4. Right-click Script Rules and select Create Default Rules.
5. Right-click Packaged app Rules and select Create Default Rules.

> [!WARNING]
> Do not skip the Packaged app Rules. In Windows 11 and Server 2025, critical interface elements like the Settings app, Search, and the Start Menu are Packaged apps. If you enforce AppLocker without creating the default Packaged app rules, the entire desktop interface will fail to load for standard users.

## 3. Creating the Custom Deny Rule

Once the safety net of the default rules is in place, you can create your specific restrictions. In this example, we will block WordPad.

**Configuration Steps:**

1. Right-click Executable Rules and select Create New Rule.
2. Action: Select Deny.
3. User or group: Leave as Everyone (or target a specific non-admin group).
4. Conditions: Select Path.
5. Path: Enter `%PROGRAMFILES%\Windows NT\Accessories\wordpad.exe`
6. Name: "Deny WordPad execution" and click Create.

## 4. Enforcement Configuration (Audit First)

Never deploy AppLocker directly into Enforce mode. Always use Audit mode first to verify that your rules are not accidentally blocking legitimate business applications or background system processes.

**Configuration Steps:**

1. Right-click the root AppLocker node and select Properties.
2. Under the Enforcement tab, check Configured for Executable rules.
3. Select Audit only from the dropdown. (Repeat for Packaged apps, Scripts, etc.)
4. Apply the GPO to your test OU.

> [!IMPORTANT]
> After running in Audit only mode for several days and reviewing the Event Viewer logs for false positives, you can return to this properties menu and switch the dropdown to Enforce rules.

## 5. Troubleshooting

Verifying AppLocker involves reviewing the local event logs to see what the policy is actively blocking (or would block, if in Audit mode) and understanding how to recover a machine if a bad policy is applied.

### 5.1 Reviewing AppLocker Logs

AppLocker does not log its actions to the standard System or Application logs. You must navigate to its dedicated operational log to see execution blocks.

**Log Location (Event Viewer):** `eventvwr.msc` ; Navigate to: `Applications and Services Logs > Microsoft > Windows > AppLocker`

**What to look for:**

- EXE and DLL log: Look for Event ID 8003 (Warning) if in Audit Mode. This means "This application WOULD have been blocked."
- Look for Event ID 8004 (Error) if in Enforce Mode. This means "This application WAS blocked."
- Review the event details to identify exactly which file path or publisher was caught by the policy. If a critical Windows 11 background process is listed here, you must create a Permit exception before switching to Enforce mode.

### 5.2 Testing a File Against Local Policies

You can use PowerShell to test if a specific executable will be allowed or blocked by the currently active AppLocker policy on a machine.

**Command:** `Get-AppLockerPolicy -Effective | Test-AppLockerPolicy -Path "C:\Program Files\Windows NT\Accessories\wordpad.exe" -User "Everyone"`

**What it checks and variables to look for:**

- This command simulates the execution of the file.
- It will return `Allowed` or `Denied` along with the specific `PolicyDecision` rule that triggered the result.

### 5.3 Emergency Recovery (Bricked System)

If you accidentally enforce a policy without default rules and cannot log into Windows or open the MMC to remove the GPO, you must manually destroy the local AppLocker cache.

**Recovery Steps:**

1. Boot the machine into Windows Recovery Environment (WinRE) or Safe Mode with Command Prompt.
2. Navigate to `C:\Windows\System32\AppLocker`
3. Delete all `.applocker` files in this directory.
4. Reboot the machine. This forces Windows to clear the strict enforcement state, allowing you to log in, correct the bad GPO, and run `gpupdate /force`.

<!-- Created by: Gergő Téringer, 2026 -->