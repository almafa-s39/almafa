<!-- 
---
title: "DNS"
author: "Gergő Téringer"
---
 -->
# DNS

This document provides administrative guidance for configuring Domain Name System (DNS) services on Windows Server 2025 and Windows 11 enterprise environments. It covers zone management, zone types, DNS record creation, standard forwarders, and conditional forwarders using both PowerShell cmdlets and Graphical User Interface (GUI) methods.

> [!NOTE]
> Management of Windows DNS requires elevated administrative privileges (Domain Admin or local Administrator) and the `DnsServer` PowerShell module installed via RSAT or native server roles.

## 1. DNS Zones and Zone Types

A DNS zone represents a distinct, contiguous portion of the global domain name space hosted on a DNS server. Windows Server DNS supports several zone types depending on infrastructure requirements and Active Directory integration.

- **Primary Zone**: The master copy of the zone database. Reads and writes occur directly on this server.
- **Secondary Zone**: A read-only copy of a primary zone located on a separate server, updated via zone transfers for load balancing and redundancy.
- **Stub Zone**: A lightweight zone containing only authoritative records (SOA, NS, and glue A records) necessary to identify authoritative servers for a remote domain.
- **Active Directory-Integrated Zone**: A primary zone stored directly within Active Directory Domain Services (AD DS). Zone data automatically replicates to other Domain Controllers across the domain or forest using AD replication, removing single points of failure.

> [!IMPORTANT]
> Storing zones in Active Directory provides secure multi-master updates, automated secure dynamic DNS (DDNS) updates, and encrypted replication traffic without requiring manual zone transfer configuration.

### 1.1 Creating DNS Zones via PowerShell

```PowerShell
# Create an Active Directory-Integrated Primary Forward Lookup Zone
Add-DnsServerPrimaryZone -Name "corp.contoso.com" `
                         -ReplicationScope "Domain" `
                         -DynamicUpdate "Secure"

# Create a Secondary Zone pointing to a master DNS server
Add-DnsServerSecondaryZone -Name "branch.contoso.com" `
                           -MasterServers "192.168.10.50" `
                           -ZoneFile "branch.contoso.com.dns"

# Create a Stub Zone pointing to an external authoritative DNS server
Add-DnsServerStubZone -Name "partner.local" `
                      -MasterServers "10.200.1.10" `
                      -ZoneFile "partner.local.dns"
```

**Command Breakdown & Explanation:**

- `Add-DnsServerPrimaryZone`: Provisions a new primary DNS lookup zone on the targeted server.
- `-Name "corp.contoso.com"`: Defines the fully qualified domain name (FQDN) for the new zone.
- `-ReplicationScope "Domain"`: Integrates the zone into AD DS and replicates it across all domain controllers in the local domain.
- `-DynamicUpdate "Secure"`: Restricts dynamic record registration solely to authenticated domain members to prevent rogue DNS spoofing.
- `Add-DnsServerSecondaryZone`: Configures a read-only secondary copy of an existing zone.
- `-MasterServers`: Specifies the IP address of the primary server sending zone transfer updates.
- `Add-DnsServerStubZone`: Configures a lightweight zone referencing authoritative name servers for another domain.

### 1.2 Creating DNS Zones via GUI

1. Open **DNS Manager** by executing `dnsmgmt.msc` from the Run dialog (`Win + R`).
2. Expand the targeted DNS server node, right-click **Forward Lookup Zones**, and select **New Zone**.
3. In the **New Zone Wizard**, click **Next**.
4. Select the desired zone type (**Primary zone**, **Secondary zone**, or **Stub zone**). To store in Active Directory, check **Store the zone in Active Directory (available only if domain controller is a domain controller)**.
5. Choose the AD replication scope (e.g., **To all DNS servers running on domain controllers in this domain**).
6. Enter the **Zone Name** (e.g., `corp.contoso.com`).
7. Select the Dynamic Update policy (recommended: **Allow only secure dynamic updates**).
8. Review the summary page and click **Finish**.

## 2. DNS Resource Records Management

DNS resource records map human-readable domain names to network identifiers and services. Common record types include:

- **A Record**: Maps a host FQDN to an IPv4 address.
- **AAAA Record**: Maps a host FQDN to an IPv6 address.
- **CNAME Record**: Alias record pointing a canonical name to another canonical host FQDN.
- **MX Record**: Mail Exchanger record specifying mail servers responsible for receiving email for the domain.
- **PTR Record**: Pointer record mapping an IP address back to an FQDN in Reverse Lookup Zones.
- **SRV Record**: Service Locator record identifying servers hosting specific domain services (e.g., Kerberos, LDAP).

### 2.1 Adding DNS Records via PowerShell

```PowerShell
# Create an A record with automatic PTR creation
Add-DnsServerResourceRecordA -ZoneName "corp.contoso.com" `
                            -Name "appserver01" `
                            -IPv4Address "10.10.1.100" `
                            -CreatePtr

# Create a CNAME alias pointing to the application server
Add-DnsServerResourceRecordCName -ZoneName "corp.contoso.com" `
                                -Name "portal" `
                                -HostNameAlias "appserver01.corp.contoso.com"

# Create an MX record with priority ranking
Add-DnsServerResourceRecordMX -ZoneName "corp.contoso.com" `
                              -Name "." `
                              -MailExchange "mail.contoso.com" `
                              -Preference 10
```

**Command Breakdown & Explanation:**

- `Add-DnsServerResourceRecordA`: Adds a host (A) mapping record to the specified zone.
- `-ZoneName "corp.contoso.com"`: Target forward lookup zone hosting the record.
- `-Name "appserver01"`: Host identifier short name (resolves to `appserver01.corp.contoso.com`).
- `-IPv4Address "10.10.1.100"`: Assigns the IPv4 destination address.
- `-CreatePtr`: Automatically provisions a corresponding reverse lookup PTR record if the reverse lookup zone exists.
- `Add-DnsServerResourceRecordCName`: Adds an alias record pointing one host name to another existing FQDN.
- `Add-DnsServerResourceRecordMX`: Defines mail routing priority (`-Preference`) for incoming domain mail traffic.

### 2.2 Adding DNS Records via GUI

1. Launch **DNS Manager** (`dnsmgmt.msc`).
2. Expand **Forward Lookup Zones** and select the target zone (e.g., `corp.contoso.com`).
3. Right-click inside the zone pane and select the record type to create:
   - For host records: Select **New Host (A or AAAA)**.
   - For alias records: Select **New Alias (CNAME)**.
   - For mail routing: Select **New Mail Exchanger (MX)**.
4. Input the **Name** and destination target (**IP Address** or **FQDN**).
5. Check **Create associated pointer (PTR) record** if adding a host record and reverse lookup is configured.
6. Click **Add Host** or **OK**.

## 3. Forwarders and Conditional Forwarders

Forwarders dictate how a local DNS server processes queries for domain names outside its own authoritative zones.

- **Standard Forwarders**: Global upstream DNS servers (e.g., corporate central DNS, `1.1.1.1`, or `8.8.8.8`) used to resolve any external domain queries that cannot be answered locally.
- **Conditional Forwarders**: Rules targeting specific external domain names (e.g., `partnercompany.com`) to route queries directly to the partner's authoritative DNS servers over a VPN or private connection, avoiding public root hint lookups.

> [!TIP]
> Use conditional forwarders for cross-domain trusts, B2B integrations, and hybrid cloud topologies (such as Azure Private DNS zones) to avoid unnecessary internet query traversals.

### 3.1 Managing Forwarders and Conditional Forwarders via PowerShell

```PowerShell
# Set global upstream DNS forwarders
Set-DnsServerForwarder -IPAddress "1.1.1.1", "8.8.8.8" `
                       -UseSPIP $true

# Create a Conditional Forwarder for a specific partner domain
Add-DnsServerConditionalForwarderZone -Name "partnercompany.com" `
                                      -MasterServers "192.168.200.10", "192.168.200.11" `
                                      -ReplicationScope "Domain"
```

**Command Breakdown & Explanation:**

- `Set-DnsServerForwarder`: Replaces or updates the server-level global forwarder list.
- `-IPAddress "1.1.1.1", "8.8.8.8"`: List of upstream recursive resolvers.
- `-UseSPIP $true`: Uses root hints if configured forwarders become unresponsive.
- `Add-DnsServerConditionalForwarderZone`: Creates a domain-specific forwarding rule.
- `-Name "partnercompany.com"`: Target domain name routed explicitly.
- `-MasterServers`: Authoritative IP addresses for the target domain.
- `-ReplicationScope "Domain"`: Replicates the conditional forwarder rule across all domain controllers automatically.

### 3.2 Managing Forwarders and Conditional Forwarders via GUI

**Configuring Global Forwarders:**

1. In **DNS Manager** (`dnsmgmt.msc`), right-click the root DNS server node and select **Properties**.
2. Navigate to the **Forwarders** tab and click **Edit**.
3. Type the IP addresses of upstream DNS servers (e.g., `1.1.1.1`) and click **OK**.
4. Click **Apply** and then **OK**.

**Configuring Conditional Forwarders:**

1. In **DNS Manager**, right-click the **Conditional Forwarders** folder and select **New Conditional Forwarder**.
2. Enter the **DNS Domain** (e.g., `partnercompany.com`).
3. Click the IP addresses table and enter the target remote DNS server addresses.
4. Optionally check **Store this conditional forwarder in Active Directory** and select the replication scope.
5. Click **OK**.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate DNS resolution, zone health, and forwarder responsiveness using native diagnostic cmdlets and utilities in PowerShell on Windows 11 or Windows Server 2025.

### 4.1 Verify DNS zone status and properties

**Command:** `Get-DnsServerZone -Name "corp.contoso.com" | Select-Object ZoneName, ZoneType, IsDsIntegrated, IsPaused, DynamicUpdate`

**What it checks and variables to look for:**

- **ZoneName**: Must be `corp.contoso.com`
- **ZoneType**: Must be `Primary`
- **IsDsIntegrated**: Must be `True`
- **IsPaused**: Must be `False`
- **DynamicUpdate**: Must be `Secure`

### 4.2 Verify DNS record resolution and authoritative mapping

**Command:** `Resolve-DnsName -Name "appserver01.corp.contoso.com" -Type A`

**What it checks and variables to look for:**

- **Name**: Must be `appserver01.corp.contoso.com`
- **Type**: Must be `A`
- **IPAddress**: Must match `10.10.1.100`
- **QueryStatus**: Must be `Success`

### 4.3 Verify DNS global forwarders configuration and responsiveness

**Command:** `Get-DnsServerForwarder | Select-Object IPAddress, UseRootHints, Timeout`

**What it checks and variables to look for:**

- **IPAddress**: Must contain target addresses such as `1.1.1.1` and `8.8.8.8`
- **UseRootHints**: Must be `True`
- **Timeout**: Default must be `3` (seconds)

### 4.4 Verify conditional forwarder resolution for external partners

**Command:** `Test-DnsServer -IPAddress 127.0.0.1 -Context Forwarder`

**What it checks and variables to look for:**

- **Result**: Must be `Success`
- **TcpStatus**: Must be `Success`
- **UdpStatus**: Must be `Success`

<!-- Created by: Gergő Téringer, 2026 -->