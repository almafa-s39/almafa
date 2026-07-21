<!-- 
---
title: "Strongswan Route based IKEv1 PSK"
author: "Gergő Téringer"
---
 -->
# Strongswan Route based IKEv1 PSK

## /etc/network/interfaces

Create a GRE tunnel

```bash
auto ipsec0
iface ipsec0 inet static
    address 10.255.255.1/30
    pre-up ip tunnel add ipsec0 mode gre local 1.1.1.1 remote 2.2.2.2
    up ip link set ipsec0 up
    down ip link set ipsec0 down
    post-down ip tunnel del ipsec0 mode gre local 1.1.1.1 remote 2.2.2.2
```

## Strongswan setup

### Install packages

```bash
apt install -y strongswan-swanctl charon-systemd
```

### Create strongswan configuration

Create a file with .conf extension under the `/etc/swanctl/conf.d` directory with the following content

```bash
connections {
    s2s {
        local_addrs=1.1.1.1
        remote_addrs=2.2.2.2
        local {
            auth=psk
            id=1.1.1.1
        }
        remote {
            auth=psk
            id=2.2.2.2
        }
        children {
            net {
                local_ts=dynamic[gre]
                remote_ts=dynamic[gre]
                mode=transport
                start_action=start|trap
                updown=/script/updown.sh
            }
        }
    version=1
    }
}

secrets {
    ike1{
        id=1.1.1.1
        id=2.2.2.2
        secret="Skill39$$"
    }
}
```

The content of `/script/udpown.sh`:

```bash
#!/bin/bash

if [ $PLUTO_VERB == 'up-host' ]; then
    ip route add 10.10.20.0/24 dev ipsec0 via 10.255.255.2
fi

if [ $PLUTO_VERB == 'down-host' ]; then
    ip route del 10.10.20.0/24 dev ipsec0 via 10.255.255.2
fi
```

<!-- Created by: Gergő Téringer, 2026 -->