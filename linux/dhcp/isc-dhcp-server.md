# ISC-DHCP-SERVER

> [!NOTE]
> This package is deprecated, so it is recommend to use KEA-DHCP4-SERVER, because it is the one that being developed. (If you don't want to use advanced features it works well)
> This configuration will include DDNS as well, but you just leave those lines out if you want to make it without DDNS.

## Install packages

```bash
apt install isc-dhcp-server
```

## Create listen

Edit `/etc/default/isc-dhcp-server` configuration file, and add the interfaces to listen on.

## Create DDNS key

Create a tsig key, for updating your zone (included in bind9 package):

```bash
tsig-keygen "ddns" > /etc/dhcp/ddns.key
```

## Configure the service

Enter `/etc/dhcp/dhcpd.conf` and edit the following lines:

```bash
include "/etc/dhcp/ddns.key";
ddns-update-style standard;

subnet 10.10.10.0 mask 255.255.255.0 {
    range 10.10.10.100 10.10.10.199;
    option routers 10.10.10.254;
    option domain-name-servers 10.10.20.10, 10.10.20.11;
    zone unitel.com. {
        primary 10.10.20.10
        key "ddns";
    }
    zone 20.10.10.in-addr.arpa. {
        primary 10.10.20.10
        key "ddns";
    }
    ddns-domainname "unitel.com.";
    ddns-rev-domainname "in-addr.arpa";
}
```

## Failover peering

Create a new tsig-key to make it secure

```bash
tsig-keygen "omapi-key" > /etc/dhcp/omapi.key
```

Add these lines to configuration to make it work

```bash
include "/etc/dhcp/omapi.key"
omapi-port 7911;
omapi-key "omapi-key";

failover peer "failover" {
    primary/secondary;

    # BOTH
    address <peering_address>;
    peer_address <peer_address>;
    load balance max seconds 3;

    # Primary
    mclt 3600;
    split 128;
}

subnet ... {
    # ..
    pool {
        range 10.10.10.100 10.10.10.199;
        failover peer "failover";
    }
    # ..
}
```
