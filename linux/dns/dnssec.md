<!-- 
---
title: "dnssec"
author: "Gergő Téringer"
---
 -->
# DNSSEC

This document details administrative procedures for configuring DNS Security Extensions (DNSSEC) on a BIND9 DNS server running on Debian 13 (Trixie). It covers package installation, key generation (Zone Signing Keys and Key Signing Keys), zone signing, root hint definition, trust anchor deployment, and chain-of-trust verification.

> [!NOTE]
> DNSSEC adds cryptographic signatures to existing DNS records to prevent domain spoofing and cache poisoning attacks.

## 1. Package Installation and Prerequisites

> [!IMPORTANT]
> A functional primary BIND9 DNS server hosting at least one authoritative domain zone must be operational before deploying DNSSEC key signing structures.

```bash
# Install BIND9 core binaries, documentation, and DNSSEC management utilities
apt install bind9 bind9-doc bind9utils dnsutils
```

**Command Breakdown & Explanation:**

- `apt install bind9 bind9-doc bind9utils dnsutils`: Installs the core BIND9 daemon, documentation files, administration utilities (such as `dnssec-keygen` and `dnssec-signzone`), and diagnostic CLI tools (`dig` and `delv`).

## 2. Server Options and Root Hints Configuration

To enable DNSSEC validation globally across the daemon, modify `/etc/bind/named.conf.options`.

```Bash
# Enable DNSSEC validation in /etc/bind/named.conf.options
# Ensure the following parameter is set inside the options { ... }; block:
# dnssec-validation yes;
```

Next, define the root hint zone inside `/etc/bind/named.conf.local` pointing to a local root hints file.

```Bash
# Append root zone hint definition to /etc/bind/named.conf.local
cat << 'EOF' >> /etc/bind/named.conf.local

zone "." IN {
    type hint;
    file "/var/cache/bind/root.hint";
};
EOF
```

Create the `/var/cache/bind/root.hint` file to define authoritative root servers.

```Plaintext
3600000 IN NS ca.isp.net.
ca.isp.net. 3600000 A 1.214.51.1
```

**Command Breakdown & Explanation:**

- `dnssec-validation yes;`: Enforces cryptographic validation on DNS responses received by the resolver.
- `zone "." IN { type hint; ... }`: Configures BIND9 to utilize a custom root hint file (`root.hint`) for resolving root-level delegation queries.

## 3. Cryptographic Key Generation and Zone Signing

DNSSEC requires two distinct key pairs:

- **Zone Signing Key (ZSK)**: Cryptographically signs standard RRsets (A, AAAA, MX, etc.) within the zone.
- **Key Signing Key (KSK)**: Signs the DNSKEY RRset containing the ZSK to establish a secure chain of trust.

```Bash
# Navigate to the BIND cache directory where keys are stored
cd /var/cache/bind

# Generate a Zone Signing Key (ZSK)
dnssec-keygen -a NSEC3RSASHA1 -b 2048 -n ZONE <domain>

# Generate a Key Signing Key (KSK)
dnssec-keygen -f KSK -a NSEC3RSASHA1 -b 4096 -n ZONE <domain>
```

**Command Breakdown & Explanation:**

- `cd /var/cache/bind`: Moves into BIND's designated working directory where keys and signed zone files reside.
- `dnssec-keygen -a NSEC3RSASHA1 -b 2048 -n ZONE <domain>`: Generates a 2048-bit RSA key pair utilizing NSEC3-capable SHA-1 hashing to serve as the ZSK.
- `dnssec-keygen -f KSK ...`: Generates a 4096-bit RSA key pair flagged explicitly as a Key Signing Key (`-f KSK`).

Include both generated `.key` files inside your un-signed zone file (e.g., `/var/cache/bind/unitel.db`).

```Plaintext
$INCLUDE Kunitel.com.+007+26268.key
$INCLUDE Kunitel.com.+007+48554.key
```

```Bash
# Cryptographically sign the target zone file
dnssec-signzone -L 3600 -A -3 $(head -c 1000 /dev/random | sha1sum | cut -b 1-16) -N INCREMENT -o <domain> -t <zone_file>
```

**Command Breakdown & Explanation:**

- `dnssec-signzone`: Processes the un-signed zone file, signs all record sets with the included key pairs, and generates a `<zone_file>.signed` output file.
- `-L 3600`: Sets the default Time-To-Live (TTL) for generated DNSSEC records (RRSIG, NSEC3) to 3600 seconds.
- `-A`: Includes NSEC3PARAM records in the signed output.
- `-3 $(head -c 1000 /dev/random | sha1sum | cut -b 1-16)`: Specifies NSEC3 opt-out execution using a 16-character pseudo-random salt string.
- `-N INCREMENT`: Automatically increments the SOA serial number of the signed zone.
- `-o <domain>`: Defines the origin domain name.
- `-t <zone_file>`: Specifies the source un-signed zone file.

## 4. Activating Signed Zones and Trust Anchor Provisioning

Modify `/etc/bind/named.conf.local` to direct BIND9 to load the newly created `.signed` zone file.

```Plaintext
zone "unitel.com" IN {
    type master;
    file "/var/cache/bind/unitel.db.signed";
    allow-update { key rndc-key; };
};
```

```Bash
# Restart BIND9 to load the signed zone configuration
systemctl restart bind9
```

To complete the DNSSEC chain of trust across parent TLD and Root servers:

1. Executing `dnssec-signzone` automatically outputs a `dsset-<domain>` file containing the Delegation Signer (DS) records.
2. Copy `dsset-<domain>` to the parent Top-Level Domain (TLD) server and append its contents to the TLD zone file.
3. On the TLD/Root server, generate ZSK and KSK key pairs, sign the TLD zone, and submit the TLD DS records to the Root zone file.
4. On domain servers and client resolvers, edit `/etc/bind/bind.keys` to insert the trusted Root Server key:

```Plaintext
trust-anchors {
    initial-key 257 37 "AwEAAem3CdaN2tJyY64XCt2KFzTTu74ccqo4YzBMmsdkFIa1pqoK4JtZ...";
};
```

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate DNSKEY records, RRSIG cryptographic signatures, and the DNSSEC trust chain on Debian 13 using standard diagnostic utilities.

### 5.1 Verify DNSKEY record availability

**Command:** `dig DNSKEY <domain> @localhost`

**What it checks and variables to look for:**

- **status**: Must be `NOERROR`
- **ANSWER SECTION**: Must display `2` DNSKEY records (ZSK and KSK)

### 5.2 Verify RRSIG signature records

**Command:** `dig A <domain> @localhost +noadditional +dnssec`

**What it checks and variables to look for:**

- **flags**: Must include `ad` or `aa` flags
- **ANSWER SECTION**: Must return both the requested `A` record and its corresponding `RRSIG` signature record

### 5.3 Verify DNSSEC chain of trust using delv

**Command:** `delv <domain> +multi +rtrace`

**What it checks and variables to look for:**

- **Validation Output**: Must output `; fully validated`
- **Fetch Chain**: Must display sequential queries for `<domain>/A`, `<domain>/DNSKEY`, `<domain>/DS`, `tld/DNSKEY`, `tld/DS`, and `./DNSKEY`

<!-- Created by: Gergő Téringer, 2026 -->