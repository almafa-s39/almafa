<!-- 
---
title: "Postfix Auto-Responder (Echo Service)"
author: "Gergő Téringer"
---
 -->

# Postfix Auto-Responder (Echo Service)

This document provides administrative procedures for configuring an automated email responder (Echo Service) using Postfix on Debian 13 (Trixie). It utilizes Postfix's `pipe` transport daemon to intercept incoming messages directed to a specific address and automatically reply with a predefined template.

> [!NOTE]
> This configuration assumes the target echo address (`echo@dmz.worldskills.org`) exists as a valid mailbox or destination within your environment (such as an LDAP backend or local alias map) so that Postfix accepts the incoming message before routing it to the custom transport script.

## 1. Master Configuration (master.cf)

To define a custom handling pipeline for the auto-responder, you must append a new transport definition to the Postfix master configuration file.

```Bash
# Append the autoreply transport definition to /etc/postfix/master.cf
cat << 'EOF' >> /etc/postfix/master.cf

autoreply unix  -       n       n       -       -       pipe
  flags=Rq user=administrator argv=/etc/postfix/scripts/autoreply.sh $sender
EOF
```

**Command Breakdown & Explanation:**

- `autoreply unix ... pipe`: Defines a new custom transport named `autoreply` that utilizes the Postfix `pipe` daemon to hand off incoming message payloads to an external script.
- `flags=Rq`: Passes the envelope recipient and sender parameters safely, quoting arguments as necessary.
- `user=administrator`: Specifies the unprivileged or dedicated system user account that will execute the shell script.
- `argv=...`: Points directly to the execution script and passes the sender's address (`$sender`) as the first argument (`$1`).

## 2. Autoreply Script and Message Template Creation

Create a dedicated directory to store the automation script and the plain-text response template, ensuring appropriate permissions are granted to the executing user.

```Bash
# Create the script and template directory
mkdir -p /etc/postfix/scripts
```

### 2.1 Create the Response Body Template (thank-you.txt)

Create the text file containing the email headers and body that will be sent back to the original sender.

```Bash
cat << 'EOF' > /etc/postfix/scripts/thank-you.txt
Subject: Thank you!

Thank you for reaching out.
EOF
```

### 2.2 Create the Execution Script (autoreply.sh)

Create the automation script that calls `sendmail` to dispatch the reply.

```Bash
cat << 'EOF' > /etc/postfix/scripts/autoreply.sh
#!/bin/bash
/usr/sbin/sendmail -F "Echo" -f "echo@dmz.worldskills.org" $1 < /etc/postfix/scripts/thank-you.txt
EOF

# Grant execute permissions to the script
chmod +x /etc/postfix/scripts/autoreply.sh
```

**Command Breakdown & Explanation:**

- `/usr/sbin/sendmail`: The standard mail submission interface interface used to inject outbound messages back into the mail queue.
- `-F "Echo"`: Sets the human-readable sender display name for the outgoing reply.
- `-f "echo@dmz.worldskills.org"`: Sets the envelope sender address.
- `$1`: Represents the sender address passed dynamically by the Postfix `pipe` transport from the incoming message.
- `< /etc/postfix/scripts/thank-you.txt`: Pipes the response template into `sendmail` as the email body.

## 3. Transport Map Configuration

You must create a transport map table to instruct Postfix to route emails destined specifically for the echo address through your newly created `autoreply` transport instead of standard delivery.

```Bash
# Create the transport map file and register the mapping
cat << 'EOF' > /etc/postfix/transport
echo@dmz.worldskills.org autoreply:
EOF

# Compile the plain-text file into a binary hash database for Postfix
postmap /etc/postfix/transport
```

### 3.1 Register Transport Map in main.cf

Configure Postfix to load the compiled transport map by updating the main configuration file.

```Bash
# Append the transport map configuration to /etc/postfix/main.cf
cat << 'EOF' >> /etc/postfix/main.cf

# Custom Transport Maps
transport_maps = hash:/etc/postfix/transport
EOF

# Restart the Postfix service to apply all configuration updates
systemctl restart postfix
```

**Command Breakdown & Explanation:**

- `echo@dmz.worldskills.org autoreply:`: Forces any message addressed to this exact user to bypass normal mail storage and invoke the `autoreply` transport service defined in `master.cf`.
- `postmap`: Compiles the text-based mapping table into a high-performance indexed database (`.db`) readable by Postfix.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the Postfix transport configuration and test the auto-responder on Debian 13 using standard administrative tools and log monitors.

### 4.1 Verify Postfix configuration syntax

**Command:** `postfix check`

**What it checks and variables to look for:**

- **Error Output**: Must execute silently without returning fatal path errors, missing lookup maps, or syntax typos in `main.cf` or `master.cf`.

### 4.2 Verify live mail logs during a test transmission

**Command:** `tail -f /var/log/postfix.log` *(or check `journalctl -u postfix` depending on your logging backend)*

**What it checks and variables to look for:**

- **Pipe Execution**: When sending a test email to `echo@dmz.worldskills.org`, the logs must show Postfix successfully accepting the message, matching the `transport_maps` entry, and invoking the `autoreply` pipe command, followed by queuing the outbound response.

<!-- Created by: Gergő Téringer, 2026 -->