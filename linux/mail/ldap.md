<!-- 
---
title: "Mail with LDAP authentication"
author: "Gergő Téringer"
---
 -->
# Mail with LDAP authentication

## Pre-eliminary settings

You have to have users with the following attributes in LDAP:

```ldif
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
```

## Postfix

Install following packages:

- postfix
- postfix-ldap

### LDAP lookups

You have to create 3 files for setup which will do LDAP queries
The basic part will be used in every part:
```
version = 3
server_host = ldap://ldap.example.net

ldap_search = dc=example,dc=net
ldap_scope = sub

query_filter = (|(mail=%s)(uid=%s))
```

`/etc/postfix/ldap/vbox`

```
result_attribute = uid
result_format = /mailboxes/%s/
```

`/etc/postfix/ldap/vuid`

```
result_attribute = uidNumber
```

`/etc/postfix/ldap/vgid`

```
result_attribute = gidNumber
```

### /etc/postfix/master.cf

Uncomment the following lines

```
submissions inet    n   -   y   -   -   smtpd
 # ...
    -o smtpd_tls_wrappermode=yes
    -o smtpd_sasl_auth_enable=yes
 # ...
```

### /etc/postfix/main.cf

```
mydomain=unitel.com

# Add certificate and key into the configuration
smtpd_tls_{key/cert}_file = /path/to/your/files

mydestination = localhost

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

virtual_mailbox_base = /
virtual_mailbox_domains = example.net

virtual_mailbox_maps = ldap:/etc/postfix/ldap/vbox
virtual_uid_maps = ldap:/etc/postfix/ldap/vuid
virtual_gid_maps = ldap:/etc/postfix/ldap/vgid

maillog_file = /var/log/postfix.log
```

## Dovecot

Install following packages:

- dovecot-imapd
- dovecot-core
- dovecot-ldap

Dovecot will be seperated under `/etc/dovecot/conf.d/` directory
Edit the following files

`10-auth.conf`

```
auth_allow_cleartext = no # Require SSL
auth_mechanisms = plain login

#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```

`10-mail.conf`

```
# Uncomment lines where mail attributes are and add the following lines

mail_driver = maildir
mail_path = /mailboxes/%{user}
```

`10-master.conf`

```
# Uncomment imaps ports (or pop3s ports if you plan to deploy that that).
service auth {
    unix_listener /var/spool/postfix/private/auth {
        user=postfix
        group=postfix
        mode=0666
    }
}
```

Adjust certificates in this file `10-ssl.conf` to the path or yours.

Place `auto = subscribe` into every mailbox attribute in `15-mailboxes.conf`

`auth-ldap.conf.ext`

```
ldap_uris = ldap://ldap.example.net
ldap_auth_dn = cn=admin,dc=example,dc=net
ldap_auth_dn_password = Skill39$$
ldap_base = dc=example,dc=net
ldap_scope = subtree

passdb ldap {
    ldap_filter = (&(objectClass=posixAccount)(uid=%{user}))
    ldap_bind = yes
    default_password_scheme = SSHA
    fields {
        user=%{ldap:uid}
        mail_uid=%{ldap:uid}
        userdb_home=%{ldap:homeDirectory}
        userdb_uid=%{ldap:uidNumber}
        userdb_gid=%{ldap:gidNumber}
    }
}

userdb ldap {
    ldap_filter = (&(objectClass=posixAccount)(uid=%{user}))
}
```

<!-- Created by: Gergő Téringer, 2026 -->