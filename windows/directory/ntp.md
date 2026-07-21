<!-- 
---
title: "Network Time Protocol (NTP) Configuration"
author: "Gergő Téringer"
---
 -->
# Network Time Protocol (NTP) Configuration

Maintaining accurate and synchronized time across a Windows domain is critical. Active Directory relies on the Kerberos authentication protocol, which strictly requires the clocks of all communicating machines to be within 5 minutes of each other. If time drifts beyond this threshold, authentication fails, and users lose access to domain resources.

In a standard Active Directory environment, time follows a strict hierarchy: domain members sync with local Domain Controllers, and local Domain Controllers sync with the Domain Controller holding the **PDC Emulator** FSMO role.

This GPO configuration specifically defines how the PDC Emulator (or an isolated domain environment) reaches out to a reliable external time source.

> [!WARNING]  
> **Targeting:** You should apply this specific GPO *only* to the Domain Controller holding the PDC Emulator role (often done using a WMI filter). If you apply an explicit "NTP" type configuration to all standard domain-joined computers, you will break the native AD domain hierarchy (NT5DS) and create widespread synchronization issues.

## 1. Windows Time Service Configuration

These policies dictate how the Windows Time service (`w32time`) operates, where it pulls time from, and whether it answers time requests from other clients.

`Computer Configuration > Policies > Administrative Templates > System > Windows Time Service > Time Providers`

### 1.1 Configure Windows NTP Client

> [!NOTE]  
> Set **Configure Windows NTP Client** to `Enabled` and populate the following fields:
>
> - **NtpServer:** `time.windows.com,0x8` (Replace with your preferred reliable FQDN or IP, such as `pool.ntp.org,0x8`).
> - **Type:** `NTP`
> - **CrossSiteSyncFlags:** `2`
> - **ResolvePeerBackoffMinutes:** `15`
> - **ResolvePeerBackoffMaxTimes:** `7`
> - **SpecialPollInterval:** `3600` (1 hour)
> - **EventLogFlags:** `0`

**Parameter Breakdown:**

- **NtpServer Flag (`0x8`):** The hexadecimal flag appended to the FQDN is critical. `0x8` instructs the Windows Time service to send the time request in strict "Client" mode. Other common flags include `0x1` (SpecialInterval) or `0x9` (Client mode + SpecialInterval).
- **Type (`NTP`):** Setting this to `NTP` forces the machine to query the specified external servers. (Standard domain members use `NT5DS` by default, which tells them to find a DC automatically).
- **SpecialPollInterval:** Determines how often (in seconds) the machine polls the external server. `3600` ensures it checks every hour.

### 1.2 Enable Windows NTP Client

> [!NOTE]  
> Set **Enable Windows NTP Client** to `Enabled`.
> This ensures the local machine's internal clock actively accepts and processes the time data retrieved from the server defined in step 1.1.

### 1.3 Enable Windows NTP Server

> [!NOTE]  
> Set **Enable Windows NTP Server** to `Enabled`.
> Because this machine acts as the time authority for the rest of the network (e.g., standard workstations, servers, and network switches), it must be allowed to answer inbound NTP requests on UDP port 123.

---

## 2. Troubleshooting

Verifying NTP involves checking the operational status of the `w32time` service, forcing a synchronization, and querying where the machine currently believes it should get its time.

### 2.1 Verifying Current Time Source and Status

Use this command to see a comprehensive overview of the local time service, including the stratum layer and the active reference server.

```powershell
w32tm /query /status
```

**What it checks and variables to look for:**

- `Leap Indicator`: Should be `0 (no warning)`. If it says `3 (not synchronized)`, the service has failed to reach the external server.
- `Stratum`: Displays the distance from the reference clock. A PDC Emulator syncing to the internet usually sits at Stratum `2` or `3`.
- `Source`: Must display the FQDN or IP you configured in the GPO (e.g., `pool.ntp.org,0x8`). If it says `Local CMOS Clock` or `Free-running System Clock`, the GPO has not applied or the external source is blocked by a firewall.

### 2.2 Verifying GPO Application to the Service

Sometimes the GPO applies to the registry, but the Windows Time service hasn't picked up the new parameters. Check the actual configuration active in memory.

**Command:** `w32tm /query /configuration`

**What it checks and variables to look for:**

- Look under the **[TimeProviders]** section. Ensure **Type** is set to `NTP` and the **NtpServer** string exactly matches your GPO definition.

### 2.3 Forcing a Resync

If the configuration is correct but the time is still drifting, you can manually trigger the service to discover its peers and fetch an immediate update.

**Command:** `w32tm /resync /rediscover`

**What it checks and variables to look for:**

- If successful, it will return `The command completed successfully.`
- If it returns `The computer did not resync because no time data was available`, verify that outbound UDP Port `123` is open on your perimeter firewall, as many ISPs and enterprise firewalls block outbound NTP traffic by default.

<!-- Created by: Gergő Téringer, 2026 -->