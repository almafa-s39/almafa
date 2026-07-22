<!-- 
---
title: "Email PAM Authentication and LMTP"
author: "Gergő Téringer"
---
 -->
# Email PAM Authentication and LMTP

This document provides the administrative procedures for setting up an email server using Postfix and Dovecot with PAM (Pluggable Authentication Modules) authentication. It includes Local Mail Transfer Protocol (LMTP) integration, configuration breakdowns, and verification test cases to ensure the services are operating securely and correctly.

> [!NOTE]
> By implementing LMTP, Postfix will no longer write emails directly to the disk. Instead, it will hand the emails off to Dovecot via a UNIX socket, allowing Dovecot to natively handle the final delivery.

## 1. System Setup

Install the Postfix and Dovecot packages, including the specific LMTP daemon. Choose "internet site" when dpkg asks for the configuration type. Next, create the mailboxes folder with sufficient permissions.

```Bash
apt install postfix dovecot-core dovecot-imapd dovecot-lmtpd -y
mkdir /mailboxes/ && chmod 777 /mailboxes/
```

**Command Breakdown & Explanation:**

- `apt install postfix dovecot-core dovecot-imapd dovecot-lmtpd -y`: Installs the core Mail Transfer Agent (Postfix), the IMAP/Authentication daemon (Dovecot), and the Dovecot LMTP delivery module.
- `mkdir /mailboxes/`: Creates the root directory where all user mail will be stored.
- `chmod 777 /mailboxes/`: Grants read, write, and execute permissions to all users for this directory, ensuring the mail daemons can create individual user folders dynamically.

## 2. Postfix Configuration

### 2.1 master.cf (Listening Ports)

You must configure the port settings in `/etc/postfix/master.cf`. The first section is for TCP/587 (StartTLS), and the second part is for TCP/465 (TLS). You just have to uncomment the following lines:

```Plaintext
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_relay_restrictions=
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject

submissions inet n      -       y       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_relay_restrictions=
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

**Configuration Breakdown & Explanation:**

- `submission` / `submissions`: Opens port 587 (StartTLS) and port 465 (Implicit TLS).
- `-o smtpd_sasl_auth_enable=yes`: Enables Simple Authentication and Security Layer (SASL), requiring clients to authenticate before sending mail.
- `-o smtpd_tls_wrappermode=yes`: Forces immediate TLS encryption upon connection on port 465.
- `-o smtpd_recipient_restrictions=permit_sasl_authenticated,reject`: Prevents the server from acting as an open relay; only authenticated users are permitted to send emails to external domains.

### 2.2 main.cf (Core Settings and LMTP Routing)

Edit the `mydomain` and `myorigin` variables in `/etc/postfix/main.cf` if they aren't your domain or hostname, or what you have to call your mailserver. Add your network subnet(s) to the end of `mynetworks`. Add your domain to the end of `mydestination`.

Edit the certificate paths and configurations to encrypt the traffic and use your certificate and key. Finally, add the last configuration lines to use Dovecot as the authentication authority, route local mail to the LMTP socket, and define where to log.

```Ini, TOML
myorigin = /etc/mailname
mydomain = globex-isp.com
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 0.0.0.0/0
mydestination = $myhostname, ispsrv.globex-isp.com, localhost.globex-isp.com, localhost, globex-isp.com

# SMTP server RSA key and certificate in PEM format
smtpd_tls_key_file = /ca/isp/server.key
smtpd_tls_cert_file = /ca/isp/server.crt
smtpd_tls_security_level = encrypt
smtp_tls_CAfile = /ca/certs/bundleCA.crt
smtp_tls_security_level = encrypt
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

myhostname = ispsrv.globex-isp.com
inet_interfaces = all

# Add to the end of the file
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

# Instruct Postfix to hand off local mail to Dovecot's LMTP socket
mailbox_transport = lmtp:unix:private/dovecot-lmtp

maillog_file = /var/log/postfix.log
```

**Configuration Breakdown & Explanation:**

- `smtpd_tls_security_level = encrypt`: Mandates that all incoming connections must be encrypted with TLS.
- `smtpd_sasl_type = dovecot`: Instructs Postfix to hand off authentication requests to Dovecot.
- `smtpd_sasl_path = private/auth`: Defines the relative path to the UNIX socket where Dovecot is listening for authentication queries.
- `mailbox_transport = lmtp:unix:private/dovecot-lmtp`: Tells Postfix to stop local delivery and instead push messages to Dovecot's LMTP socket. The path is relative to the Postfix chroot jail (`/var/spool/postfix/`).

After saving, restart Postfix and check the logs using `tail -f /var/log/postfix.log`.

## 3. Dovecot Configuration

Make these minimal configurations to the files listed below. All files are located under the `/etc/dovecot/conf.d/` directory.

### 3.1 10-auth.conf

Configure the following variables to use plain and encrypted login, and use System login.

```Ini, TOML
auth_allow_cleartext = no
auth_mechanisms = plain login
!include auth-system.conf.ext
```

**Configuration Breakdown & Explanation:**

- `auth_allow_cleartext = no`: Prevents clients from sending passwords in plain text unless the connection is already secured by SSL/TLS.
- `!include auth-system.conf.ext`: Activates the PAM (Pluggable Authentication Modules) backend, allowing Dovecot to authenticate users against the local Linux system accounts (e.g., `/etc/shadow`).

### 3.2 10-mail.conf

Configure which mailbox path with driver to use. Uncomment those 4 lines and use the 2 below.

```Ini, TOML
mail_driver = maildir
mail_path = /mailboxes/%{user}
```

**Configuration Breakdown & Explanation:**

- `mail_driver = maildir`: Uses the Maildir format (one file per email) rather than the legacy mbox format (one large file for all emails).
- `mail_path = /mailboxes/%{user}`: Dynamically resolves to the specific user's folder inside the root mailboxes directory created during system setup.

### 3.3 10-master.conf

Configure the IMAP listener, the LMTP socket for mail delivery, and the `private/auth` socket to allow Postfix to authenticate.

```Ini, TOML
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

service auth {
  unix_listener auth-userdb {
  }
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

**Configuration Breakdown & Explanation:**

- `port = 143` / `port = 993`: Explicitly binds the IMAP and secure IMAPS listeners to their default ports.
- `unix_listener .../dovecot-lmtp`: Creates the UNIX socket inside the Postfix chroot jail that allows Dovecot to receive incoming mail directly from Postfix.
- `unix_listener .../private/auth`: Creates the socket file inside Postfix's chroot jail that allows Postfix to securely pass client credentials to Dovecot for verification.

### 3.4 10-ssl.conf

Configure the following SSL settings with your certificate paths.

```Ini, TOML
ssl = required
ssl_server_cert_file = /ca/isp/server.pem
ssl_server_key_file = /ca/isp/server.key
ssl_server_ca_file = /ca/bundleCA.crt
```

```Bash
# Restart Dovecot to apply all configurations and create the LMTP socket
systemctl restart dovecot
```

## 4. Verification and Test Cases

> [!NOTE]
> Validate the configuration syntax, service state, active listening sockets, and authentication flow on Debian 13 using standard administrative tools.

### 4.1 Verify Service Status and Listening Ports

**Command:** `ss -tulnp | grep -E 'master|dovecot'`

**What it checks and variables to look for:**

- **Postfix (master)**: Must display `LISTEN` on `*:25` (SMTP), `*:587` (Submission), and `*:465` (Submissions).
- **Dovecot**: Must display `LISTEN` on `*:143` (IMAP) and `*:993` (IMAPS).

### 4.2 Verify the LMTP Socket

**Command:** `ss -xl | grep dovecot-lmtp`

**What it checks and variables to look for:**

- **Output**: Must display the active UNIX socket at `/var/spool/postfix/private/dovecot-lmtp`, confirming Dovecot is successfully listening for Postfix handoffs.

### 4.3 Verify Postfix and Dovecot Configuration Syntax

**Command (Postfix):** `postfix check`
**Command (Dovecot):** `doveconf -n > /dev/null`

**What it checks and variables to look for:**

- **Output**: Both commands must execute silently. Any output indicates a syntax error, a missing configuration file, or an unreadable certificate path that needs to be addressed.

### 4.4 Test Case: Verify Dovecot PAM Authentication (IMAP)

**Command:** `openssl s_client -connect 127.0.0.1:993 -quiet`
*(Once connected, type: `a1 LOGIN your_linux_username your_password`)*

**What it checks and variables to look for:**

- **TLS Handshake**: Must connect successfully and display the server certificate chain.
- **Authentication**: Must return `a1 OK [CAPABILITY...] Logged in` indicating that Dovecot successfully read the `/etc/shadow` file via PAM.

### 4.5 Test Case: Verify Postfix SASL Authentication (SMTP)

**Command:** `openssl s_client -connect 127.0.0.1:465 -quiet`
*(Once connected, type: `EHLO localhost`)*

**What it checks and variables to look for:**

- **SASL Capability**: The server must return `250-AUTH PLAIN LOGIN`, verifying that Postfix is successfully communicating with the Dovecot auth socket defined in `master.cf`.

<!-- Created by: Gergő Téringer, 2026 -->