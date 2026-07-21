<!-- 
---
title: "SSSD"
author: "Gergő Téringer"
---
 -->
# SSSD

## User authentication

The LDAP users need to show up in the system as 'Unix' users for the authentication and permissions to work properly. This will be done with SSSD.

First, install the package:

```bash
apt install sssd-ldap
cp /usr/share/doc/sssd-common/examples/sssd-example.conf /etc/sssd/sssd.conf
```

Now, create and edit `/etc/sssd/sssd.conf`:

```ini
[sssd]
config_file_version = 2
domains = domain.com
default_domain_suffix = domain.com

[domain/domain.com]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://127.0.0.1
cache_credentials = True
ldap_search_base = dc=domain,dc=com
```

Set permissions to this file, then restart the service.

```bash
chmod 0600 /etc/sssd/sssd.conf
chown root:root /etc/sssd/sssd.conf
systemctl restart sssd
```

<!-- Created by: Gergő Téringer, 2026 -->