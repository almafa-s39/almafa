# Time

Maintaining accurate and synchronized time across a network infrastructure is critical. Without proper time synchronization, syslog timestamps become unreliable for troubleshooting, cryptographic certificates may fail validation, and time-based access control lists (ACLs) will not trigger correctly.

## Timezone

By default, Cisco IOS devices operate in Coordinated Universal Time (UTC). Configuring the local timezone and daylight saving time (DST) ensures that local console messages and logs reflect the correct regional time.

```cisco
clock timezone CET 1 0
clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
```

### Command Breakdown & Explanation

- `clock timezone CET 1 0`:
  - `CET`: Sets the acronym for the timezone (Central European Time).
  - `1 0`: Sets the offset from UTC. In this case, it is +1 hour and 0 minutes.
- `clock summer-time CEST recurring...`:
  - `CEST`: Sets the acronym for daylight saving time (Central European Summer Time).
  - `recurring`: Instructs the router to automatically apply this change every year.
  - `last Sun Mar 2:00`: Defines the exact start time for DST (the last Sunday in March at 02:00).
  - `last Sun Oct 3:00`: Defines the exact end time for DST (the last Sunday in October at 03:00).

**Practical Example:** When you issue a show logging command during a routing flap in July, the timestamps will accurately show CEST rather than defaulting to UTC, saving you the mental math of converting times during a critical troubleshooting session.

---

## NTP

NTP ensures that all devices in the topology share the exact same clock. To prevent rogue devices from injecting incorrect time into the network, NTP authentication is highly recommended.

### NTP Master (Server)

An NTP Master acts as the authoritative time source for other devices in the network. This is often configured on a core router or a designated boundary router when a reliable external internet time source is unavailable or isolated.

```cisco
ntp master 5
ntp authenticate
ntp authentication-key 1 md5 Passw0rd!
ntp trusted-key 1
```

**Command Breakdown & Explanation:**

- `ntp master 5`: Configures the router to act as an NTP server. The 5 represents the Stratum level. Stratum 1 is a direct atomic clock; by setting this to 5, you leave room for this router to potentially sync to a more authoritative source (like a Stratum 2 or 3 internet server) later without causing stratum loops.
- `ntp authenticate`: Enables the NTP authentication feature globally on the device.
- `ntp authentication-key 1 md5 Passw0rd!`: Creates an MD5 hashing key. 1 is the key ID, and Passw0rd! is the shared secret string.
- `ntp trusted-key 1`: Instructs the router that key ID 1 is trusted. Without this, the router will know the key but will not actually use it to validate time requests.

### NTP Client

The NTP client synchronizes its clock to the upstream Master. It must be configured with the exact same authentication parameters to successfully peer with the server.

```cisco
ntp authenticate
ntp authentication-key 1 md5 Passw0rd!
ntp trusted-key 1
ntp server 30.30.30.30 key 1
```

**Command Breakdown & Explanation:**

- `ntp authenticate` / `authentication-key` / `trusted-key`: These commands mirror the Master configuration, enabling authentication and defining the shared secret so the client can verify the server's identity.
- `ntp server 30.30.30.30 key 1`: Points the client to the IP address of the NTP Master (`30.30.30.30`). The `key 1` suffix is crucial; it tells the client to use the specific trusted key we defined to authenticate the NTP packets coming from this specific server.

**Practical Example:** If you are deploying a new branch distribution switch, applying this client configuration ensures it securely pulls its time from the core router (30.30.30.30), preventing man-in-the-middle attacks from spoofing the time and invalidating the switch's local certificates.

## Troubleshooting

Verifying time and NTP configuration involves checking the local hardware clock, ensuring that the NTP synchronization process has successfully completed, and confirming that authentication is passing between peers.

### Verifying Local Clock and Timezone

This command confirms that your manual timezone offset and daylight saving time configurations are active and currently applied to the router's internal clock.

**Command:** `show clock detail`

What it checks and variables to look for:

- `Time and Date`: Displays the current system time (e.g., `14:35:00.123 CEST Mon Jul 15 2026`).
- `Time source`: Indicates where the router is getting its time. You want to see `Time source is NTP`. If it says `hardware calendar` or `user configuration`, NTP is not functioning properly.
- `Summer time`: Confirms if DST is currently active or inactive based on your configured recurring schedule.

### Verifying NTP Synchronization Status

This is the primary command to determine if the router has successfully locked onto an NTP server and updated its system clock.

**Command:** `show ntp status`

What it checks and variables to look for:

- `Clock is`: Must show `synchronized`. If it shows `unsynchronized`, the router has not yet established a reliable connection with the NTP server (this can take up to 10-15 minutes after configuration).
- `Stratum`: Shows the local router's stratum level. For a client connected to a Stratum 5 master, this should read `6`.
- `Reference`: Displays the IP address of the server that this device is currently synchronized to.

### Verifying NTP Peers and Authentication

If the clock remains unsynchronized, this command provides a detailed view of the communication between the client and the configured NTP servers, highlighting reachability and authentication failures.

**Command:** `show ntp associations`

What it checks and variables to look for:

- `address`: The IP address of the configured NTP server.
- `ref clock`: The upstream source your server is synced to. If this shows `127.127.1.1`, the master is using its own local hardware clock (common for isolated `ntp master` setups).
- `st`: The stratum of the server. A stratum of `16` means the server is considered unreachable or invalid.
- `reach`: An octal counter that tracks the success of the last eight NTP polls. A value of `377` indicates 100% reachability. A value of `0` means the server is not responding to ping/NTP requests.
- `*` (Asterisk): Look for an asterisk next to the server address. This indicates it is the currently selected, active system peer.
- `~` (Tilde): If you see a tilde instead of an asterisk, it often indicates an authentication failure (e.g., mismatched MD5 key).
