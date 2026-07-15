# SNMP

Simple Network Management Protocol (SNMP) is essential for monitoring network performance, hardware health, and interface statistics. The following documentation covers configurations for SNMPv1, SNMPv2c, and SNMPv3, alongside best practices like access control and trap generation.

## 1. Administrative details

Before configuring polling or traps, it is a best practice to define the physical location and the administrative contact for the device.

**Configuration:**

```cisco
snmp-server contact network-admin@company.local
snmp-server location Rack 42, Server Room 1
```

**Explanation:**
These commands populate the metadata of the router, which is automatically queried by Network Management Systems (NMS) to organize inventory databases.

---

## 2. Access Control (security)

Regardless of the SNMP version used, it is highly recommended to use an Access Control List (ACL) to restrict which IP addresses are allowed to poll the device or receive traps.

**Configuration:**

```cisco
ip access-list standard SNMP-NMS-ACL
 permit 10.20.200.100
```

---

## 3. SNMPv1 Configuration

SNMPv1 is the legacy version of the protocol. It uses plain-text community strings for authentication and offers no encryption. It is generally not recommended for modern networks unless required by legacy polling systems.

**Configuration:**

```cisco
snmp-server community PublicRO RO SNMP-NMS-ACL
snmp-server community PrivateRW RW SNMP-NMS-ACL
```

**Command Breakdown & Explanation:**

- `snmp-server community PublicRO RO`: Creates a Read-Only (RO) plain-text community string named `PublicRO`. The NMS can only view data.
- `snmp-server community PrivateRW RW`: Creates a Read-Write (RW) community string named `PrivateRW`. The NMS can view and alter device configurations.
- `SNMP-NMS-ACL`: Binds the previously created ACL to the community string, ensuring only the NMS at `10.20.200.100` can use these credentials.

===

## 4. SNMPv2c Configuration

SNMPv2c improves upon v1 by adding support for bulk data retrieval (GetBulk) and Informs (acknowledged traps). However, it still uses plain-text community strings and lacks encryption, making it vulnerable to packet sniffing.

**Configuration:**

```cisco
snmp-server community WSC-Monitor RO SNMP-NMS-ACL
```

**Command Breakdown & Explanation:**

- `snmp-server community WSC-Monitor RO`: Defines the community string `WSC-Monitor` with Read-Only privileges. The configuration syntax is identical to v1, but the NMS will use v2c protocol features (like GetBulk) when polling.
- `SNMP-NMS-ACL`: Applies the security filter to restrict access.

---

## 5. SNMPv3 Configuration

SNMPv3 provides significant security enhancements over v1 and v2c by introducing the AuthPriv model, ensuring that management traffic is both authenticated (identity verified) and encrypted (data privacy).

**Configuration:**

```cisco
snmp-server view WSC-VIEW iso included
snmp-server group WSC-GRP v3 priv read WSC-VIEW access SNMP-NMS-ACL
snmp-server user wscmon WSC-GRP v3 auth sha Passw0rd! priv aes 128 Passw0rd!
```

**Command Breakdown & Explanation:**

- `snmp-server view WSC-VIEW iso included`: Creates a view named WSC-VIEW that includes the entire MIB tree root (iso), allowing full read access.
- `snmp-server group WSC-GRP v3 priv read WSC-VIEW access SNMP-NMS-ACL`: Creates a group (WSC-GRP) enforcing authentication and encryption (priv). It binds the WSC-VIEW for read access and limits access using the SNMP-NMS-ACL.
- `snmp-server user wscmon WSC-GRP v3 auth sha Passw0rd! priv aes 128 Passw0rd!`: Creates the user 'wscmon' in the group. It sets the authentication protocol to SHA and the privacy (encryption) protocol to 128-bit AES with their respective passwords.

---

## 6. SNMP Traps and Informs

While SNMP polling allows the NMS to ask the router for data, Traps and Informs allow the router to proactively alert the NMS when an event occurs. Informs are more reliable than Traps because they require an acknowledgment from the NMS.

**Configuration:**

```cisco
snmp-server enable traps
snmp-server trap-source Loopback0
snmp-server source-interface informs Loopback0
snmp-server host 10.20.200.100 version 3 priv wscmon config snmp
```

**Command Breakdown & Explanation:**

- `snmp-server enable traps`: Globally enables the router to generate trap messages for system events.
- `snmp-server trap-source` / `source-interface informs Loopback0`: Forces the router to use the IP address of Loopback0 as the source IP for all outgoing alerts, ensuring a stable source IP even if physical interfaces flap.
- `snmp-server host 10.20.200.100 version 3 priv wscmon config snmp`: Defines the NMS destination (10.20.200.100). It instructs the router to send fully encrypted v3 traps using the `wscmon` credentials, specifically for configuration and general SNMP events.
