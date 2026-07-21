<!-- 
---
title: "Bind9"
author: "Gergő Téringer"
---
-->
# Bind9

## Packages

```shell
apt install bind9 bind9-utils bind9-doc
```

## Options

### Default options to include

These options are mandatory on a competition to work from the least effort, and these were default options in Bookworm, but in Trixie, the're not even in the `/etc/bind/named.conf.options` file so you have to add them.

```bash
options {
    # Enable recursion and allow it to acl.
    recursion yes;
    allow-recursion { any; };
    allow-query-cache { any; };
    
    dnssec-validation no;

    forwarderse {
        1.1.1.1;
        8.8.8.8;
    };
};
```

### Logging

```bash
logging {
    channel query {
        file "/var/lib/bind/query.log";
        print-time yes;
        print-category yes;
        print-severity yes;
        severity debug;
    };

    channel default {
        file "/var/lib/bind/all.log";
        print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
    };

    category queries { query; };
    category default { default; };
};
```

## `/etc/bind/named.conf`

In the main configuration you just include the other configuration files to use. If you're using views, don't include named.conf.root-hints, or configure views in that file as well! For the configurations for servers I will use `/etc/bind/named.conf.local`, but create a new file and include it in this file if you want to do so!

## Primary server configuration

> [!NOTE]
> This configuration will include two  views to show how you can sepereate the lookups by source ip. I will show here how to configure zone transfer and update, but if you use the same configuration for all zones, i would place the configuration in the `/etc/bind/named.conf.options` file in the options directive, and it will apply to all of your zones.

```bash
# To generate the following key
# tsig-keygen "update" > /etc/bind/update.key
include "/etc/bind/update.key" 

acl intra { 10.0.0.0/8; };
acl mydns { 
    192.168.100.101;
    192.168.100.102;
    192.168.100.103;
};

acl mydhcp { 
    10.0.0.1;
};

view inside {
    match-clients { intra; };
    zone company.com {
        type master;
        # Use this path if you set up DDNS and /etc/bind/ if you want static files.
        file "/var/lib/bind/company.com.in"; 
        allow-transfer { "mydns"; };
        also-notify { "mydns"; };
        allow-update { "mydhcp"; key "update"; };
    };
};

view outside {
    match-clients { any; };
    zone company.com {
        type master;
        # Use this path if you set up DDNS and /etc/bind/ if you want static files.
        file "/var/lib/bind/company.com.ex"; 
        allow-transfer { "mydns"; };
        also-notify { "mydns"; };
    };
};
```

## Secondary server configuration

```bash
include "/etc/bind/update.key" 

acl intra { 10.0.0.0/8; };
acl mydns { 
    192.168.100.100;
}

view inside {
    match-clients { intra; };
    zone company.com {
        type slave;
        file "/var/lib/bind/company.com.in"; 
        masters { "mydns"; };
    };
};

view outside {
    match-clients { any; };
    zone company.com {
        type slave;
        file "/var/lib/bind/company.com.ex"; 
        masters { "mydns"; };
    };
};
```

## Forwarder server configuration (conditional forwarding)

```bash
zone google.com {
    type forward;
    forwarders {
        8.8.8.8;
    };
};
```

## Reverse zone configuration

If you have to configure reverse zones, follow this naming

```bash
# for 10.20.30.0/24
zone 30.20.10.in-addr.arpa {
    # ...
};

# for 2001:db8:1010:1010::/64
zone 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa {
    # ...
};
```

## Records

### A

DNS name to IPv4 mapping.

```bash
<NAME> 	A   <IPv4>
www     A   10.10.10.10
```

### AAAA

DNS name to IPv6 mapping.

```bash
<NAME> 	AAAA    <IPv6>
www     AAAA    2001:db8:1010::1010
```

### CNAME

DNS name to DNS name mapping. It ends with a dot!!!

```bash
<NAME>    CNAME   <TARGET>
web       CNAME   www.domain.name.
```

### NS

You have to configure an A or AAAA record where the NS record points!

```bash
<NAME> 			NS <TARGET>
domain.name	    NS ns1.domain.name.
```

### SOA

It marks the beginning of a DNS zone and identifies the zone's authoritative properties. REQUIRED for all zones!

- **MNAME**: Primary nameserver (DDNS server if applicable)
- **RNAME**: Admin contact email name
- **Serial**: Zone version (Important for zone transfers)
- **Refresh**: Delay between checks
- **Retry**: Delay after failure
- **Expire**: If the server doesn't get queries by this time (seconds), it will be marked as non authoritative
- **Minimum**: Negative caching global TTL

```bash
@   SOA   <TARGET>.         <EMAIL>.                 ( <SERIAL> <REFRESH> <RETRY> <EXPIRE> <MINUMUM> )
@   SOA   ns1.domain.name.  your\.mail.domain.name.  ( 1 1h 5m 1d 5m )
```

### PTR

Reverse DNS lookup, from IP to DNS. It can only target A or AAAA records!

```bash
# Usage
<REMAINING_ADDRESS>                 PTR  <TARGET>.

# Examples
# IPv4 - 30.20.10.in-addr.arpa
10                                  PTR  www.domain.name.

# IPv6 - 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa
0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.1     PTR  www.domain.name.
```

### MX

You have to configure an A or AAAA record where the MX record points!

```bash
<NAME> 			MX <PRIORITY>   <TARGET>
domain.name	    MX 10           mail.domain.name
```

### SRV

```bash
_<service>._<proto>.<NAME>			SRV <PRIORITY> <WEIGHT> <PORT> <TARGET>
_minecraft._tcp.mc1.domain.name 	SRV 10          0       25565   srv4.domain.name.
```

## Zone file

### Apparmor

You can place your Bind configuration files into `/etc/named/` or `/var/lib/bind/`. If you want to place it elsewhere, you will get *permission denied* if it has 777 as well! You have to configur Apparmor, to make the daemon able to read or write from or into that directory and subfiles and folders. Create and edit `/etc/apparmor.d/local/usr.sbin.named`!

`/etc/apparmor.d/local/usr.sbin.named`

```bash
/storage/dns/ r,
/storage/dns/** rw,
```

After this configuration /it can read the directory `/storage/dns/` and read and write everyting under it.

### File setup

#### By hand

Create a zone files including SOA and NS record.

##### Forward lookup zone - domain.name

```bash
$TTL 1d
$ORIGIN domain.name.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.
@   MX      10  ns.domain.name.

ns  A       10.20.30.10
ns  AAAA    2001:db8:1010:1010::1010
www CNAME   ns.domain.name.
```

##### IPv4 Reverse lookup zone - 30.20.10.in-addr.arpa

```bash
$TTL 1d
$ORIGIN domain.name.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.

10  PTR     ns.domain.name.
```

##### IPv6 Reverse lookup zone - 0.1.0.1.0.1.0.1.8.b.d.0.1.0.0.2.ip6.arpa

```bash
$TTL 1d
$ORIGIN domain.name.
@   SOA     ns.domain.name. admin.domain.name.  ( 1 12h 5m 1d 5m )
@   NS      ns.domain.name.

0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.1     PTR  ns.domain.name.
```

#### SOA from file

If you're lazy, or just want to use a file as db.empty (like in Bookworm or earlier versions), just us this command, and you get file with SOA record. Install *bind9-doc* to achieve this!

```bash
grep -A 11 "; default TTL for zone" /usr/share/doc/bind9-doc/arm/chapter3.html | awk -F'</span>' '{print $2}' > /etc/bind/db.empty
```

<!-- Created by: Gergő Téringer, 2026 -->