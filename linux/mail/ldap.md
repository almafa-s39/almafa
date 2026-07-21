<!-- 
---
title: "Mail with LDAP authentication"
author: "Gergő Téringer"
---
 -->
# Postfix and Dovecot Mail Server with LDAP Authentication Guide

This document provides administrative procedures for configuring a complete Mail Transfer Agent (MTA) and Mail Delivery Agent (MDA) using Postfix and Dovecot on Debian 13 (Trixie). It integrates OpenLDAP to centrally authenticate users and map virtual mailboxes, utilizing TLS encryption and SASL for secure client connections.

> [!NOTE]
> Postfix handles the routing and receiving of SMTP mail, while Dovecot manages local mail storage (IMAP/POP3) and provides the SASL authentication socket that Postfix relies on to verify users against the LDAP directory.

## 1. Directory Prerequisites (OpenLDAP)

Before configuring the mail server, ensure your LDAP directory contains users populated with the required `inetOrgPerson` and `posixAccount` object classes.

An example LDIF structure for a valid mail user:

```LDIF
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
userPassword: {SSHA}YourHashedPasswordHere # slappswd -s >> <file>.ldif
```

## 2. Postfix Installation and LDAP Maps

Install the Postfix MTA and the LDAP extension module required to query the directory.

```Bash
# Install Postfix and the LDAP lookup module
apt install postfix postfix-ldap

# Create the directory to house the LDAP query maps
mkdir -p /etc/postfix/ldap
```

### 2.1 Postfix LDAP Query Maps

Postfix requires three separate lookup files to determine the virtual mailbox path, the User ID (UID), and the Group ID (GID) based on the recipient's email address or LDAP username.

**Virtual Mailbox Map (`/etc/postfix/ldap/vbox`):**

```Ini, TOML
server_host = ldap://ldap.example.net
version = 3
ldap_search = dc=example,dc=net
ldap_scope = sub
query_filter = (|(mail=%s)(uid=%s))
result_attribute = uid
result_format = /mailboxes/%s/
```

**Virtual UID Map (`/etc/postfix/ldap/vuid`):**

```Ini, TOML
server_host = ldap://ldap.example.net
version = 3
ldap_search = dc=example,dc=net
ldap_scope = sub
query_filter = (|(mail=%s)(uid=%s))
result_attribute = uidNumber
```

**Virtual GID Map (`/etc/postfix/ldap/vgid`):**

```Ini, TOML
server_host = ldap://ldap.example.net
version = 3
ldap_search = dc=example,dc=net
ldap_scope = sub
query_filter = (|(mail=%s)(uid=%s))
result_attribute = gidNumber
```

**Command Breakdown & Explanation:**

- `query_filter`: Instructs Postfix to search the LDAP directory matching either the exact `mail` attribute or the `uid` attribute against the incoming address.
- `result_format`: Appends the returned `uid` into a directory path structure (e.g., `/mailboxes/username/`).

## 3. Postfix Core Configuration

Configure Postfix to utilize the LDAP maps, enable SASL authentication through Dovecot, and enforce TLS.

### 3.1 Enable Secure SMTP (Submissions)

Edit `/etc/postfix/master.cf` to enable port 465 (submissions) for encrypted client mail submission. Uncomment and modify the following lines:

```Plaintext
submissions inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

### 3.2 Main Postfix Parameters

Append or modify the following configurations in `/etc/postfix/main.cf`:

```Ini, TOML
mydomain = example.net
mydestination = localhost

# TLS Configuration
smtpd_tls_cert_file = /ca/server.crt
smtpd_tls_key_file = /ca/server.key
smtpd_use_tls = yes

# SASL Authentication (via Dovecot)
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

# Virtual Mailbox Parameters
virtual_mailbox_base = /
virtual_mailbox_domains = example.net
virtual_mailbox_maps = ldap:/etc/postfix/ldap/vbox
virtual_uid_maps = ldap:/etc/postfix/ldap/vuid
virtual_gid_maps = ldap:/etc/postfix/ldap/vgid

# Logging
maillog_file = /var/log/postfix.log
```

## 4. Dovecot Installation and Core Configuration

Dovecot will manage IMAP access for clients, handle local Maildir storage, and provide the authentication mechanism for Postfix.

```Bash
# Install Dovecot core, IMAP daemon, and LDAP module
apt install dovecot-core dovecot-imapd dovecot-ldap
```

### 4.1 Authentication Settings (/etc/dovecot/conf.d/10-auth.conf)

Disable cleartext authentication over unencrypted connections and enable the LDAP backend.

```Ini, TOML
auth_allow_cleartext = no
auth_mechanisms = plain login

#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```

### 4.2 Mail Storage Settings (/etc/dovecot/conf.d/10-mail.conf)

Define the storage driver and the physical path to the user mailboxes.

```Ini, TOML
mail_driver = maildir
mail_path = /mailboxes/%{user}
```

### 4.3 Service Socket Settings (/etc/dovecot/conf.d/10-master.conf)

Create the UNIX socket that Postfix will use to communicate with Dovecot for SASL authentication.

```Ini, TOML
service auth {
    unix_listener /var/spool/postfix/private/auth {
        user = postfix
        group = postfix
        mode = 0666
    }
}
```

### 4.4 SSL and Auto-Subscribe Settings

In `/etc/dovecot/conf.d/10-ssl.conf`, set the paths to your certificates:

```Ini, TOML
ssl = required
ssl_cert = </ca/server.crt
ssl_key = </ca/server.key
```

In `/etc/dovecot/conf.d/15-mailboxes.conf`, ensure standard folders are automatically created and subscribed for new users by adding `auto = subscribe` to the `mailbox` blocks (e.g., Drafts, Junk, Trash, Sent).

## 5. Dovecot LDAP Integration

Define how Dovecot connects to the LDAP directory to verify passwords and retrieve user metadata.

### 5.1 LDAP Authentication Config (/etc/dovecot/conf.d/auth-ldap.conf.ext)

Overwrite the file with the following parameters:

```Ini, TOML
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
        user = %{ldap:uid}
        mail_uid = %{ldap:uid}
        userdb_home = %{ldap:homeDirectory}
        userdb_uid = %{ldap:uidNumber}
        userdb_gid = %{ldap:gidNumber}
    }
}

userdb ldap {
    ldap_filter = (&(objectClass=posixAccount)(uid=%{user}))
}
```

**Command Breakdown & Explanation:**

- `ldap_bind = yes`: Forces Dovecot to attempt an actual LDAP bind using the credentials provided by the user, rather than just comparing password hashes locally.
- `passdb`: Defines how Dovecot verifies passwords.
- `userdb`: Defines how Dovecot retrieves user information (like home directories and UIDs).

## 6. Service Activation and Verification

Restart both services to apply the new configurations and create the root mailboxes directory.

```Bash
# Create the root mailboxes directory and set base permissions
mkdir -p /mailboxes
chmod 777 /mailboxes

# Restart Postfix and Dovecot
systemctl restart postfix dovecot
```

### 6.1 Verify Listening Ports

**Command:** `ss -tulnp | grep -E 'master|dovecot'`

**What it checks:**

- **Postfix**: Must be listening on `*:25` (SMTP) and `*:465` (Submissions).
- **Dovecot**: Must be listening on `*:143` (IMAP) and `*:993` (IMAPS).

### 6.2 Verify LDAP Map Parsing (Postfix)

**Command:** `postmap -q "username@example.net" ldap:/etc/postfix/ldap/vbox`

**What it checks:**

- **Output**: Must query your LDAP server and return the formatted mailbox path (e.g., `/mailboxes/username/`). If it returns nothing, check the LDAP credentials and filter in the `.cf` file.

<!-- Created by: Gergő Téringer, 2026 -->