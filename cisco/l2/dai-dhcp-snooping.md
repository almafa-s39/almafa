<!-- 
---
title: "DHCP Snooping, Dynamic ARP Inspection (DAI), and IP Source Guard"
author: "Gergő Téringer"
---
 -->
# DHCP Snooping, Dynamic ARP Inspection (DAI), and IP Source Guard

This document details the configuration of essential Layer 2 security features: DHCP Snooping, Dynamic ARP Inspection (DAI), and IP Source Guard (IPSG). These features work together to protect the local area network against rogue DHCP servers, DHCP starvation, ARP spoofing (man-in-the-middle) attacks, and IP spoofing.

> [!IMPORTANT]
> Modern Context & Compatibility: Implementing DHCP Snooping, DAI, and IPSG is critical in modern enterprise networks. Operating systems like Windows 11 heavily rely on secure and predictable DHCP/ARP operations. If a rogue DHCP server provides incorrect gateways or DNS configurations, or if an attacker spoofs a legitimate IP address, modern endpoints will fail to reach Microsoft Cloud services or local Active Directory domain controllers (like those running Windows Server 2025). Furthermore, configuring the DHCP database (`flash:dhcp.db`) ensures the switch retains the IP-to-MAC bindings across reboots, preventing valid clients from being disconnected by DAI or IPSG after a switch power cycle.

## 1. Global Configuration and Option 82 Explanation

This section initializes the DHCP Snooping database, enables the security protocols globally for specific VLANs, and enforces strict ARP packet validation.

> [!NOTE]
> The command `no ip dhcp snooping information option` explicitly disables the switch from injecting **DHCP Option 82** (Relay Agent Information) into DHCP discovery packets sent by clients. Option 82 appends physical switchport data (like switch MAC and port number) to the DHCP payload.
>
> **When do you need to configure/enable Option 82?**
>
> - You **need** Option 82 if your centralized DHCP Server (e.g., Windows Server 2025, Cisco ISE, or InfoBlox) is configured to assign specific IP addresses or deploy specific security policies based on the physical location/switchport the client is plugged into.
> - You **disable** it (as done in this script) if your DHCP server does not understand Option 82. Many default DHCP server configurations will silently drop DHCP packets that contain Option 82 data, leaving clients without an IP address.
>
> The companion command, `ip dhcp snooping information option allow-untrusted`, ensures that if a downstream edge device sends a packet that *already* contains Option 82, this switch will not drop it.

```Cisco CLI
ip dhcp snooping vlan 10
ip dhcp snooping information option allow-untrusted
no ip dhcp snooping information option
ip dhcp snooping database flash:dhcp.db
ip dhcp snooping
ip domain-name ict.pl

ip arp inspection vlan 10
ip arp inspection validate src-mac dst-mac ip
```

**Command Breakdown & Explanation:**

- `ip dhcp snooping vlan 10`: Enables DHCP snooping specifically for VLAN 10.
- `ip dhcp snooping information option allow-untrusted`: Permits the switch to accept incoming DHCP packets on untrusted ports that already have Option 82 information attached.
- `no ip dhcp snooping information option`: Prevents this switch from appending its own Option 82 data to client DHCP requests.
- `ip dhcp snooping database flash:dhcp.db`: Stores the dynamic IP-to-MAC binding table in local flash memory so the security database survives a switch reload.
- `ip dhcp snooping`: Globally enables the DHCP snooping feature on the switch.
- `ip arp inspection vlan 10`: Globally enables DAI for VLAN 10.
- `ip arp inspection validate src-mac dst-mac ip`: Enforces strict payload validation for ARP packets. It checks the source MAC, destination MAC, and the IP address in the ARP body against the Ethernet header and the DHCP snooping database, dropping packets with mismatched or malicious data.

## 2. Interface Configuration

Layer 2 security relies on a strict trust boundary. Access ports facing end-users are considered "untrusted" by default, while uplinks pointing toward the legitimate DHCP server must be explicitly trusted. IP Source Guard is applied directly to the untrusted access ports to prevent IP spoofing.

### 2.1 Access (Untrusted) Interfaces

```Cisco
interface Gig0/0
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
 spanning-tree bpduguard enable
 ip verify source
```

**Command Breakdown & Explanation:**

- `switchport access vlan 10` and `switchport mode access`: Hardcodes the port as an access port in VLAN 10. By default, because it is an access port, DHCP Snooping and DAI treat this interface as **untrusted**.
- `spanning-tree portfast edge`: Bypasses the listening and learning Spanning Tree states, bringing the port up immediately for edge devices (like PCs).
- `spanning-tree bpduguard enable`: Secures the edge port by shutting it down (err-disable) if an unauthorized switch is plugged in and sends BPDUs.
- `ip verify source`: Enables IP Source Guard on the interface. It restricts IP traffic on untrusted Layer 2 ports by filtering traffic based on the DHCP snooping binding database. If the source IP does not match the IP assigned to that port in the database, the traffic is dropped.

### 2.2 Uplink (Trusted) Interfaces

```Cisco
interface Gig1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust

interface Gig1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ip dhcp snooping trust
 ip arp inspection trust
```

**Command Breakdown & Explanation:**

- `switchport trunk encapsulation dot1q` and `switchport mode trunk`: Configures the uplink as an 802.1Q trunk port to carry multiple VLANs.
- `ip dhcp snooping trust`: Configures the interface as a trusted port. This is mandatory for uplinks facing the legitimate DHCP server, as untrusted ports will drop incoming DHCP Offer and Acknowledgment packets.
- `ip arp inspection trust`: Bypasses DAI checks for this interface, ensuring valid ARP traffic from the rest of the core/distribution network is not dropped.

## 3. Troubleshooting and Verification

### 3.1 Verify DHCP Snooping Configuration

Displays the global status of DHCP snooping, the trust state of interfaces, and Option 82 settings.

**Command:** `show ip dhcp snooping`

**Command Breakdown & Explanation:**
Validates that the security feature is active on the correct VLANs and that uplinks are properly designated as trusted.

What it checks and variables to look for:

- **Switch DHCP snooping**: Must be `Enabled`
- **Insertion of option 82**: Must be `Disabled`
- **Interface**: Your uplinks (e.g., `GigabitEthernet1/0`) must show `yes` under the `Trusted` column.

### 3.2 Verify DHCP Snooping Binding Database

Displays the dynamic IP-to-MAC bindings learned by the switch.

**Command:** `show ip dhcp snooping binding`

**Command Breakdown & Explanation:**
Checks the database to ensure end-user devices are successfully acquiring IPs and that the switch is recording their bindings for DAI and IPSG to use.

What it checks and variables to look for:

- **MacAddress**: Must match the connected client's MAC.
- **IpAddress**: Must reflect the DHCP-assigned IP.
- **State**: Must be `dhcp-snooping`.
- **Interface**: Must reflect the correct access port (e.g., `GigabitEthernet0/0`).

### 3.3 Verify Dynamic ARP Inspection

Displays the status of DAI across VLANs and logs of dropped ARP packets.

**Command:** `show ip arp inspection vlan 10`

**Command Breakdown & Explanation:**
Confirms that ARP inspection is actively protecting the VLAN and strict validation is enabled.

What it checks and variables to look for:

- **Configuration**: Must be `Enabled`.
- **Operation State**: Must be `Active`.
- **Src MAC**: Under the validation section, this must show `Enabled`.
- **Dst MAC**: Under the validation section, this must show `Enabled`.
- **IP**: Under the validation section, this must show `Enabled`.

### 3.4 Verify IP Source Guard

Displays the IP Source Guard configuration and active IP-MAC filtering bindings on access ports.

**Command:** `show ip verify source`

**Command Breakdown & Explanation:**
Confirms that IP Source Guard is actively filtering traffic on the configured access ports and correctly referencing the DHCP snooping database.

What it checks and variables to look for:

- **Interface**: Must list the configured access ports (e.g., `Gi0/0`).
- **Filter-type**: Should show `ip` (or `ip-mac` if port security integration is also enabled).
- **Status**: Must be `active`.
- **IP-address**: Must match the IP address legitimately assigned to the endpoint on that port.

<!-- Created by: Gergő Téringer, 2026 -->