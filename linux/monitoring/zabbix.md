<!-- 
---
title: "Zabbix Setup Guide"
author: "Gergő Téringer"
---
 -->
# Zabbix Setup Guide

This guide details the complete installation and configuration of Zabbix 6.0 from the Debian repository. It covers the deployment of the server, frontend, and database, as well as setting up notifications, web reachability, and SNMP monitoring.

> [!NOTE]
> Modern operating systems like Windows 11 and Windows Server 2025 have strict default security policies. When configuring Zabbix web access or SNMP polling, ensure your firewall rules explicitly allow traffic on ports 80/443 (Web), 10050 (Agent), 10051 (Server), and 161 (SNMP).

## 1. Package Installation

Install the required web server, PHP, MariaDB database, and all core Zabbix components.

```Bash
apt install apache2 libapache2-mod-php php-mysql zabbix-server-mysql zabbix-frontend-php zabbix-agent zabbix-web-service snmp
```

**Command Breakdown & Explanation:**

- `apache2 libapache2-mod-php php-mysql`: Installs the Apache web server and required PHP modules to host the Zabbix frontend.
- `zabbix-server-mysql zabbix-frontend-php`: Installs the core Zabbix server daemon and the web interface files.
- `zabbix-agent zabbix-web-service snmp`: Installs the local monitoring agent, reporting services, and SNMP tools for network polling.

## 2. Database Setup

Installing `zabbix-server-mysql` deploys a MariaDB instance. You must configure a database, create a dedicated user, and import the initial schema and data.

There are two ways to populate the database: using the provided Debian script or manually importing the SQL files.

### 2.1 Automated Script Method

Debian provides a README script that can be executed to handle the database population automatically.

> [!TIP]
> You must remove any unnecessary descriptive lines from this file, add a bash shebang (`#!/bin/bash`) to the top, and give it execute permissions before running it.

```Bash
nano /usr/share/doc/zabbix-server-mysql/README.Debian
chmod +x /usr/share/doc/zabbix-server-mysql/README.Debian
/usr/share/doc/zabbix-server-mysql/README.Debian
```

### 2.2 Manual Method

Alternatively, you can manually construct the database, user permissions, and import the required schemas.

```Bash
mysql -u root -p
```

```SQL
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Passw0rd';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;

SET GLOBAL log_bin_trust_function_creators = 1;
```

> [!CAUTION]
> The `SET GLOBAL log_bin_trust_function_creators = 1;` command is crucial. Do not omit it, otherwise importing the initial schema data will fail!

Exit MariaDB and import the SQL files in the correct order:

```Bash
cd /usr/share/zabbix/zabbix-server-mysql
zcat schema.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
zcat images.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
zcat data.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
```

Once the import is complete, log back into MariaDB to disable the global trust variable for security.

```Bash
mysql -u root -p
```

```SQL
SET GLOBAL log_bin_trust_function_creators = 0;
```

## 3. Zabbix Configuration

Link the Zabbix server configuration to the newly created database and tune the web server parameters.

```Bash
nano /etc/zabbix/zabbix_server.conf
```

```Ini
DBName=zabbix
DBUser=zabbix
DBPassword=Passw0rd
AllowUnsupportedDBVersions=1
```

Restart the Zabbix server to apply the database credentials, then enable the Apache frontend configuration.

```Bash
systemctl restart zabbix-server
a2enconf zabbix-frontend-php.conf
```

To avoid navigating to `http://<ip>/zabbix` every time, you can redirect the root directory to the Zabbix subfolder.

```Bash
nano /etc/apache2/sites-available/000-default.conf
```

```Ini
RedirectMatch ^/$ /zabbix/
```

Next, increase the PHP resource limits to accommodate Zabbix's requirements.

> [!NOTE]
> Check your exact PHP version path (e.g., `/etc/php/8.2/apache2/php.ini` or `/etc/php/8.4/apache2/php.ini` depending on your Debian release). If you forget to configure these, the Zabbix web installer will explicitly flag them as failed requirements.

```Bash
nano /etc/php/8.2/apache2/php.ini
```

```Ini, TOML
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
```

Restart Apache to load the new PHP and VirtualHost configurations.

```Bash
systemctl restart apache2
```

## 4. Running the Wizard

Navigate to `http://<zabbix-instance-ip>/zabbix` in a modern web browser to complete the visual setup wizard.

When configuring the database connection, select **MySQL** and input the credentials configured in Section 2.

> [!WARNING]
> Upon completion, the wizard will generate a configuration file (`zabbix.conf.php`) and attempt to write it to the server. If file permissions block this, the wizard will prompt you to download the file. You must manually copy it to `/etc/zabbix/web/zabbix.conf.php`.

```Bash
systemctl restart zabbix-server
```

## 5. Accessing Your Instance

After the installation wizard completes, you can log into the Zabbix dashboard using the default administrative credentials.

- **Username**: `Admin` (Case-sensitive)
- **Password**: `zabbix`

## 6. Sending Notifications

Configuring notifications ensures administrators receive alerts when services or hosts change states.

### 6.1 Media Types

> [!NOTE]
> Navigate to `Administration -> Media types`. Here you will find all preconfigured notification methods.

For email alerts, locate the **Email** or **Email (HTML)** media type. HTML provides better formatting for reading on modern clients like Outlook or Windows 11 Mail. Configure the following variables:

- **SMTP Server**: The FQDN or IP of your SMTP relay (e.g., Google, Office365, or local relay)
- **SMTP server port**: Your SMTP port (25, 465, or 587)
- **SMTP helo**: Your domain tag
- **SMTP email**: The sender email address
- **Connection security**: Match your server's SSL/TLS requirements
- **Authentication**: Set to Username and password, then fill out the credentials

> [!IMPORTANT]
> Always click **Update** at the bottom of the page to save your changes! Switch to the `Message templates` tab if you want to customize the body text of the alerts.

**Custom Scripts:**

If default media types don't fit your needs, you can trigger custom bash scripts. Click **Create media type** and set the type to **Script**.

- **Script name**: Must perfectly match the filename (e.g., `alert.sh`) placed in the `/usr/lib/zabbix/alertscripts/` directory.
- **Message template**: You must define a template, otherwise the script will receive an empty payload.

### 6.2 User Medias

Once your media type is configured, you must assign it to a user profile so Zabbix knows *where* to send the notification.

- Navigate to `Administration -> Users`.
- Select or create an administrative user.
- Go to the **Media** tab.
- Click **Add**, select your configured Media Type, and input the destination (e.g., the user's email address).
- Define the active time window and severity levels.
- Click **Update** to save the user profile.

### 6.3 Action Triggers

Actions define *when* a notification should be fired. Navigate to `Configuration -> Actions -> Trigger actions`.

- **Name**: Assign a unique, descriptive name.
- **Conditions**: Define rules (e.g., "Trigger severity is greater than or equal to Warning"). If using multiple conditions, configure the calculation type (AND/OR).
- **Enabled**: Ensure the action is checked.
- **Operations Page**: Set the default step duration (e.g., `1h`). Under Operations, define which user or user group receives the alert, and via which media type.

## 7. Web Reachability Monitoring

Zabbix can perform synthetic web checks to verify if a website is responsive and returning the correct HTTP codes.

> [!NOTE]
> Navigate to `Configuration -> Templates`. Select a template (like *HTTP service* or *HTTPS service*), then go to the `Web scenarios` tab and click **Create web scenario**.

Configure the **Scenario** tab:

- **Name**: Unique identifier for this check
- **Update interval**: Check frequency (e.g., `1m` or `30s`)
- **Attempts**: `1`
- **Agent**: `Zabbix`

Configure the **Steps** tab (Add a new step):

- **Name**: Unique step name
- **URL**: `http://fqdn`
- **Timeout**: `15s`
- **Require status codes**: `200`

> [!IMPORTANT]
> If the target web server sits behind a proxy, it is highly recommended to configure a trigger based on the HTTP response code rather than simple TCP availability.

To create a response code trigger:

- Clone an existing availability trigger.
- Append this expression using an `OR` statement: `last(/www.example.com/web.test.rspcode[Availability of www.example.com,Site availability])<>200`
- Ensure the trigger is enabled.

## 8. SNMP Monitoring (Linux)

SNMP (Simple Network Management Protocol) allows Zabbix to collect detailed metrics from network hardware and Linux/Windows servers without installing the Zabbix Agent.

### 8.1 Zabbix SNMP Host Configuration

Navigate to `Configuration -> Hosts` and click **Create host**.

- **Host name**: The display and reference name.
- **Templates**: Link an SNMP template (e.g., `Linux SNMP`).
- **Groups**: Assign the host to a logical group.
- **Interfaces**: Click Add -> SNMP. Enter the IP or DNS name. Set the port to `161` and select the SNMP version (SNMPv3 is highly recommended for modern security compliance).

For SNMPv3, populate these security fields:

| Option name | Value |
| :--- | :--- |
| **Security level** | authPriv |
| **Context name** | *(Leave empty)* |
| **Security name** | *Your configured username* |
| **Authentication protocol** | MD5 |
| **Authentication password** | *Your auth password* |
| **Privacy protocol** | AES256C |
| **Privacy password** | *Your privacy password* |

### 8.2 SNMPD Agent Configuration

On the target Linux machine being monitored, configure the SNMP daemon to accept secure v3 polling.

```Bash
apt install snmp snmpd
nano /etc/snmp/snmpd.conf

sysLocation YOURSYSTEMLOCATION
sysContact NAME <email@address>

agentaddress 0.0.0.0, [::]

createuser Administrator MD5 "Passw0rd!" AES256C "Passw0rd!"
rouser Administrator authpriv

```

Restart the SNMP daemon to apply the new user credentials and listening interfaces.

```Bash
systemctl restart snmpd
```

**Command Breakdown & Explanation:**

- `agentaddress`: Instructs the daemon to listen on all IPv4 and IPv6 interfaces.
- `createuser`: Generates an SNMPv3 user (`Administrator`) with MD5 authentication and AES-256 encryption.
- `rouser`: Grants the new user read-only access to the SNMP tree requiring `authpriv` (authentication + privacy).

## 9. Verification and Troubleshooting

> [!NOTE]
> Validate the core services are running and listening on the expected ports. If the web interface times out, ensure no host-level firewalls (like `ufw` or `iptables`) are blocking port 80.

### 9.1 Verify Zabbix Server service status

**Command:** `systemctl status zabbix-server`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/lib/systemd/system/zabbix-server.service; enabled;...)`

### 9.2 Verify web server listening state on port 80

**Command:** `ss -tuln | grep :80`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:80` or `0.0.0.0:80`

### 9.3 Verify Zabbix Server listening state on port 10051

**Command:** `ss -tuln | grep :10051`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:10051` or `0.0.0.0:10051`

<!-- Created by: Gergő Téringer, 2026 -->