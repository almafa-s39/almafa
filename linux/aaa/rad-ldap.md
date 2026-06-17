# RADIUS SQL Settings (LowBudget ISE)

## Install packages

```shell
apt install freeradius freeradius-ldap freeradius-utils
cd /etc/freeradius/3.0/
```

## Disable default site named "inner-tunnel"

```shell
rm sites-enabled/inner-tunnel
```

## Edit your default site configuration

Create `sites-enabled/default`

```shell
server radius.domain.name {
    listen {
        type = auth
        ipaddr = *
        port = 1812
    }
    
    listen {
        type = acct
        ipaddr = *
        port = 1813
    }

    authorize {
        ldap
        if ( ok || updated ) { update control { Auth-Type := ldap } }
    }

    authenticate {
        Auth-Type LDAP { ldap }
    }

    accounting { ok }
}
```

## Add your client with secret

Edit `./clients.conf` and add to the top the following part:

```shell
client openvpn_server {
 ipaddr = 10.10.10.254
 secret = Passw0rd!
 shortname = ovpn
}
```

## Restart the service

```shell
systemctl restart freeradius
```

## Latest step

### [OpenVPN setup](/linux/vpn/ovpn-rad-ldap.md)
