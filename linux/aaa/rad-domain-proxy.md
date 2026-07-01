# FreeRADIUS LDAP Domain Proxy

In this configuration there will be domain authentication against one Radius server, which will proxy the request the to another domains ( OpenVPN -> FreeRADIUS1 [example.net] if (new.example.net) then -> FreeRADIUS2 [site.example.net]). In addition in this config snippet, there will be an LDAP group based filtering, where you could add any configuration, what you want based on group membership.

> [!WARNING]
> If you want to use Rad-Pool attribute with OpenVPN, it will not work with the default `openvpn-auth-radius` package, you have to write a bash script for it!

## Pre-elminiary notes

You have to create users, and groups with the following attributes:

```ldif
# ...
dn: uid=username,ou=ou,dc=example,dc=net
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
uid: username
givenName: username
sn: username
cn: username
uidNumber: 11001
gidNumber: 11001
mail: username@example.net
homeDirectory: /home/username
loginShell: /bin/bash
userPassword: $(slappasswd -s 'YourPassword')

dn: cn=group,ou=ou,dc=example,dc=net
objectClass: posixGroup
objectClass: shadowGroup
cn: group
gidNumber: 11001
memberUid: username
# ...
```

For server 2's ldap, there will be another domain.

## FreeRADIUS Server 1 setup

Install the following packages:

- freeradius
- freeradius-ldap
- freeradius-utils

The work directory for FreeRADIUS is `/etc/freeradius/3.0/`.

Delete the `inner-tunnel` file from sites-enabled directory.

### Client configuration

Edit `clients.conf` file, add these lines to the top of the configuration

```bash
client vpn.example.net {
    ipaddr = 10.0.0.1
    secret = YourPassword
}
```

The default password for Radius login (for localhost) is `testing123`. You can change it in this file as well, just look for **secret** variable in the `client localhost { ... }` directive.

### Proxy configuration

Don't touch the default configuration just add these lines, before the realms to `proxy.conf`

```bash
home_server radius.site.example.net {
    type = auth
    ipaddr = 172.16.0.1
    port = 1812
    secret = YourPassword
}

home_server_pool pool_site {
    type = load-balance
    home_server radius.site.example.net
}
realm site.example.net {
    auth_pool = pool_site
    nostrip
}

realm example.net {
}

realm NULL {
}
```

### Site configuration

Delete the symbolic links under sites-enabled, and create a new file, includeing following parts:

```cnf
server default {
    listen {
        type=auth
        port=1812
        ipaddr=*
    }
    listen {
        type=acct
        port=1813
        ipaddr=*
    }

    authorize {
        suffix

        if (Realm == "example.net" || Realm == "NULL") {
            ldap
        }
        pap # Required bc of SSHA Password
    }

    authenticate {
        Auth-Type PAP {
            pap
        }
        Auth-Type LDAP {
            ldap
        }
    }

    post-auth {
        if (LDAP-Group == "cn=group,ou=ou,dc=example,dc=net") {
            update-reply {
                # Your content, which will be given back to your client will go here
            }
        }
    }
}
```

### LDAP configuration

Create a new symbolic link: `ln -s /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/ldap`


Edit `mods-enabled/ldap`, configure following attributes, which are already present in the configuration file:

```cfg
ldap {
    server = 'ldap://ldap.example.net'
    port = 389
    identity = 'cn=admin,dc=example,dc=net'
    password = YourPassword
    base_dn = 'dc=example,dc=net'
    # ...
    user {
        filter = "(mail=%{User-Name})"
        scope = 'sub'
    }

    group {
        scope = 'sub'
        membership_filter = .... # just comment it out
        membership_attribute = 'memberUid'
    }

    # If require TLS auth for LDAP use:
    tls {
        ca_file = /path/to/CA.crt
        require_cert = 'demand'
    }
}
```
