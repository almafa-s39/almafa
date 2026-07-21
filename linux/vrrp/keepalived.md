<!-- 
---
title: "Keepalived (FHRP)"
author: "Gergő Téringer"
---
 -->
# Keepalived (FHRP)

## Install the service using apt

```bash
apt install keepalived
```

## Primary server

Edit `/etc/keepalived/keepalived.conf`:

```bash
vrrp_script chk_haproxy {
    script "nc -zv localhost 80"
  interval 2
}

vrrp_instance VI_1 { 
    state MASTER 
    interface ens33 
    virtual_router_id 51 
    priority 100    
    advert_int 1    
    unicast_src_ip 10.1.20.21
  unicast_peer {
        10.1.20.22
    }
    authentication {
        auth_type PASS
        auth_pass Passw0rd
    }
    virtual_ipaddress {
        10.1.20.20
    }
    track_script {
        chk_haproxy
    }
}

vrrp_instance VI_2 {
    state MASTER 
    interface ens33 
    virtual_router_id 52 
    priority 100
    advert_int 1
    unicast_src_ip 2001:db8:1001:20::21
  unicast_peer {
        2001:db8:1001:20::22
    }
    authentication {
        auth_type PASS
        auth_pass Passw0rd
    }
    virtual_ipaddress {
        2001:db8:1001:20::20
    }
    track_script {
        chk_haproxy
    }
}
```

## Backup server

Edit `/etc/keepalived/keepalived.conf`:

```bash
vrrp_script chk_haproxy {
    script "nc -zv localhost 80"
  interval 2
}

vrrp_instance VI_1 { 
    state BACKUP 
    interface ens33
    virtual_router_id 51 
    priority 90
    advert_int 1
    unicast_src_ip 10.1.20.22
  unicast_peer {
        10.1.20.21
    }
    authentication {
        auth_type PASS
        auth_pass Passw0rd
    }
    virtual_ipaddress {
        10.1.20.20
    }
    track_script {
        chk_haproxy
    }
}

vrrp_instance VI_2 {
    state BACKUP 
    interface ens33 
    virtual_router_id 52 
    priority 90
    advert_int 1
    unicast_src_ip 2001:db8:1001:20::22
  unicast_peer {
        2001:db8:1001:20::21
    }
    authentication {
        auth_type PASS
        auth_pass Passw0rd
    }
    virtual_ipaddress {
        2001:db8:1001:20::20
    }
    track_script {
        chk_haproxy
    }
}
```

## Restart the service

```bash
systemctl restart keepalived
```

<!-- Created by: Gergő Téringer, 2026 -->