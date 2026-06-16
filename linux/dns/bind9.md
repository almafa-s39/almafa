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

```

## Forwarder server configuration (conditional forwarding)

```bash

```

## Records

### A

### AAAA

### CNAME

### SOA

### PTR

### MX

### SRV

## Zone file

### Apparmor

### File setup
