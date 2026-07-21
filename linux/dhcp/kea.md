<!-- 
---
title: "Kea DHCP"
author: "Gergő Téringer"
---
-->
# Kea DHCP

## Install packages

```bash
apt install kea-dhcp4-server kea-dhcp-ddns-server
```

## Create a backup and edit owners

```bash
cd /etc/kea
cp kea-dhcp4-server.conf kea4.bak
cp kea-dhcp-ddns-server.conf kea-ddns.bak
chown _kea:root ./* 
```

## Get rid off comments

```bash
grep -v "//" kea4.bak > kea-dhcp4-server.conf
grep -v "//" kea-ddns.bak > kea-dhcp-ddns-server.conf
```

## Configure kea-dhcp4

The configuration is trivial, just edit the file. If you want to add DDNS, add these three lines, somewhere in the configuration:

```bash
"dhcp-ddns": { "enable-updates": true },
"ddns-qualifying-suffix": "unitel.com",
"ddns-override-client-update": true,
```

## Configure kea-dhcp-ddns

Create a tsig key, for updating your zone (included in bind9 package):

```bash
tsig-keygen "ddns" > /etc/kea/ddns.key
```

Create a copy and make it to this form:

```bash
cp /etc/kea/ddns.key /etc/kea/ddns.json
```

`/etc/kea/ddns.json`

> [!IMPORTANT]
> Don't forget the `,` from the end because we will import it into a json file!

```json
"tsig-keys": [{
    "name": "<name>",
    "algorithm": "hmac-sha256",
    "secret": "<BASE64>"   
}],
```

Edit the configuration file, wipe out tsig-keys line and edit the following lines to achieve DDNS

```json
<?include "/etc/kea/ddns.json" ?>
"forward-ddns": {
    "ddns-domains": [{
        "name": "<domain>.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNSv4_address>" }]
    }]
},

"reverse-ddns": {
    "ddns-domains": [{
        "name": "<reverse-domain>.",
        "key-name": "<name>",
        "dns-servers": [{ "ip-address": "<DNSv4_address>" }]
    }]
},
```

<!-- Created by: Gergő Téringer, 2026 -->