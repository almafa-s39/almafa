<!-- 
---
title: "icinga"
author: "Gergő Téringer"
---
 -->

# Icinga2 Monitoring Setup

This document outlines the complete deployment and configuration of the Icinga2 monitoring core and the Icinga Web 2 interface on Debian 13 Trixie. It covers the foundational database setup, web server configuration (running on a custom port), adding Linux monitoring agents, and configuring event commands to trigger scripts on state changes.

> [!NOTE]
> Ensure that port `81` is open on your firewall. When accessed from modern desktop environments like Windows 11, Chromium-based browsers (Edge/Chrome) might warn about non-standard ports depending on network policies, but functionality will remain identical.

## 1. Core Packages and Web Server Setup

First, remove any conflicting web servers (like Apache) to prevent port bindings, then install the required Icinga2, Nginx, PostgreSQL, and PHP modules.

```Bash
# Purge Apache if it is installed
apt purge apache2*

# Install core monitoring, web, and database packages
apt install nginx php-fpm icinga2 icingaweb2 postgresql monitoring-plugins vim-icinga2 php-pgsql icingacli icingaweb2-module-monitoring

# Install the IDO database module and required PHP extensions
apt install icinga2-ido-pgsql php-ldap php-mbstring php-gd php-imagick php-mysql
```

**Command Breakdown & Explanation:**

- `apt purge apache2*`: Completely removes Apache2 and its configurations to ensure Nginx can take over without conflicts.
- `apt install ...`: Installs the Icinga2 core daemon, Icinga Web 2, the Nginx web server, PostgreSQL database, and necessary standard plugins. Note that the `-y` flag is intentionally omitted.
- `icinga2-ido-pgsql`: The database backend module allowing Icinga2 to write its status to PostgreSQL.

## 2. Nginx Web Server Configuration

Generate the Nginx configuration using the Icinga CLI, link it to the active sites, and modify it to listen on port `81` with a root redirect.

```Bash
# Generate the base Nginx configuration for Icinga Web 2
icingacli setup config webserver nginx --document-root /usr/share/icingaweb2/public > /etc/nginx/sites-available/icingaweb2.conf

# Enable the site
ln -s /etc/nginx/sites-available/icingaweb2.conf /etc/nginx/sites-enabled/

# Edit the configuration file to change the port and add a redirect
nano /etc/nginx/sites-available/icingaweb2.conf
```

Inside `/etc/nginx/sites-available/icingaweb2.conf`, find the `listen` directive and change it to `81`. Add the `location = /` block to automatically redirect the root IP address to the web interface path.

```nginx
server {
    listen 81;
    # ... existing configuration ...
    
    location = / {
        return 301 /icingaweb2;
    }
}
```

```Bash
# Restart the web services to apply changes (assuming PHP 8.4 on Debian 13)
systemctl restart nginx php8.4-fpm
```

**Command Breakdown & Explanation:**

- `icingacli setup config webserver`: Automatically builds the correct web server configuration block based on the installed paths.
- `ln -s`: Creates a symbolic link to activate the site in Nginx.
- `systemctl restart`: Bounces Nginx and the PHP-FPM socket handler to apply the new port `81` binding.

## 3. Database Initialization (PostgreSQL)

Initialize the PostgreSQL database and user credentials that Icinga Web 2 and the IDO module will use to store configuration and state data.

```Bash
# Set the password variable and create the database user and schema
ICINGAWEB2_DB_PASSWORD="Passw0rd!"
su - postgres -c "psql -c \"CREATE USER icingaweb2 WITH PASSWORD '$ICINGAWEB2_DB_PASSWORD';\""
su - postgres -c "psql -c \"CREATE DATABASE icingaweb2 OWNER icingaweb2;\""
```

**Command Breakdown & Explanation:**

- `su - postgres -c`: Runs the command as the default PostgreSQL administrative user.
- `psql -c "CREATE USER..."`: Creates the `icingaweb2` role with the defined password.
- `psql -c "CREATE DATABASE..."`: Provisions the database and assigns ownership to the newly created user.

## 4. Icinga2 Module and Wizard Preparation

Enable the necessary internal features for Icinga2, including the database backend, external commands, and the API for distributed monitoring.

```Bash
icinga2 feature enable ido-pgsql command
icingacli module enable monitoring
icinga2 api setup

# Restart Icinga2 to load the features
systemctl restart icinga2

# Start the node wizard for the master server
icinga2 node wizard
```

**Command Breakdown & Explanation:**

- `icinga2 feature enable ido-pgsql command`: Activates the PostgreSQL connection and allows accepting commands (like acknowledgments or forced checks) from the web interface.
- `icinga2 api setup`: Generates the PKI certificates required for the REST API and agent-based communication.
- `icinga2 node wizard`: Initiates the interactive setup for this machine. Configure it as a Master node.

> [!TIP]
> After configuring the backend, continue the visual setup by navigating to `http://<IP_ADDRESS>:81/icingaweb2/setup` in your browser. Create your initial admin user (e.g., `localadmin` / `Passw0rd!`) during this web wizard.

## 5. Adding Monitoring Clients (Agents)

Adding a client requires establishing trust. You will generate a ticket on the master server, install the agent on the client, and define the host objects back on the master.

> [!NOTE]
> The concepts of endpoints and zones demonstrated here apply equally whether the client is a Debian Linux node or a Windows Server 2025 machine running the Icinga 2 Windows Agent.

### 5.1 Master Node Preparation

```bash
# On the MASTER SERVER, generate a ticket for the new client
icinga2 pki ticket --cn "BR-SRV-K2.otteveny.com"
```

### 5.2 Client Node Setup

```Bash
# On the CLIENT NODE (BR-SRV-K2.otteveny.com)
apt install icinga2 monitoring-plugins
icinga2 node wizard
```

**Command Breakdown & Explanation:**

- `icinga2 pki ticket`: Generates a cryptographic token to authenticate the client to the master API without manual certificate signing.
- `icinga2 node wizard` (on client): Prompts you to connect to the master. You will enter the master's IP, port (default 5665), and the generated PKI ticket.

### 5.3 Define Host and Services on Master

Back on the master server, define the Endpoint, Zone, Host, and Service objects.

```Bash
# /etc/icinga2/conf.d/hosts.conf
/* Define the Endpoint and Zone for the secure connection */
object Endpoint "BR-SRV-K2.otteveny.com" {
}

object Zone "BR-SRV-K2.otteveny.com" {
  endpoints = [ "BR-SRV-K2.otteveny.com" ]
  parent = "master" 
}

/* Define the Host object for monitoring */
object Host "BR-SRV-K2.otteveny.com" {
  import "generic-host"
  address = "10.20.10.11" # Replace with the Client's actual IP
  
  vars.os = "Linux"
  
  # Instructs Icinga to execute checks locally on the client agent
  vars.client_endpoint = name 
  vars.web_server = "nginx"
  zone = "master"
}
```

```Bash
# /etc/icinga2/conf.d/services.conf
apply Service "http" {
  import "generic-service"
  check_command = "http"
  
  # Automatically apply this to any host tagged with Nginx
  assign where host.vars.web_server == "nginx"
}

/* Process Monitoring: Daemon Check */
apply Service "nginx-process" {
  import "generic-service"
  check_command = "procs"
  
  # Instructs the plugin to look specifically for the "nginx" process
  vars.procs_command = "nginx"
  
  # CRITICAL: Forces this specific check to execute locally on the remote agent
  command_endpoint = host.vars.client_endpoint
  
  assign where host.vars.web_server == "nginx"
}
```

## 6. Event Commands (Automated Actions)

Event commands allow Icinga2 to automatically execute a script when a host or service changes state (e.g., writing to a custom log file when a service enters a `CRITICAL` state).

### 6.1 Create the Action Script

```Bash
mkdir -p /etc/icinga2/scripts/
nano /etc/icinga2/scripts/log-error.sh
```

```Bash
#!/bin/bash
STATE=$1
STATETYPE=$2
SERVICE=$3
HOST=$4

# We only write to the log if the service hits a confirmed (HARD) CRITICAL state
if [ "$STATE" == "CRITICAL" ] && [ "$STATETYPE" == "HARD" ]; then
    echo "$(date) - ERROR: $SERVICE on $HOST is CRITICAL" >> /mnt/logs/icinga.log
fi
```

```Bash
# Make the script executable
chmod +x /etc/icinga2/scripts/log-error.sh
```

### 6.2 Define the Event Command

```Bash
# /etc/icinga2/conf.d/commands.conf
object EventCommand "log_to_file" {
  command = [ "/etc/icinga2/scripts/log-error.sh" ]

  arguments = {
    "-s" = {
      value = "$service.state$"
      skip_key = true
      order = 1
    }
    "-t" = {
      value = "$service.state_type$"
      skip_key = true
      order = 2
    }
    "-n" = {
      value = "$service.name$"
      skip_key = true
      order = 3
    }
    "-h" = {
      value = "$host.name$"
      skip_key = true
      order = 4
    }
  }
}
```

To attach this action to a service, edit the service definition in `/etc/icinga2/conf.d/services.conf` and append the event command directive.

```Bash
# Example assignment inside a service block
event_command = "log_to_file"
```

## 7. Verification and Troubleshooting

> [!IMPORTANT]
> Always validate the configuration syntax before restarting the Icinga2 daemon to avoid bringing down the monitoring system due to typos.

### 7.1 Verify Icinga2 and Nginx service status

**Command:** `systemctl status icinga2 nginx`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)` for both services.
- **Loaded**: Must be `loaded` pointing to the respective `.service` files.

### 7.2 Verify network listening state on port 81

**Command:** `ss -tuln | grep :81`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:81` or `0.0.0.0:81`

### 7.3 Validate Icinga2 configuration syntax

**Command:** `icinga2 daemon -C`

**What it checks and variables to look for:**

- **information/cli**: Must end with `Config validation done.`
- **critical/config**: Should not exist in the output. If it does, check the provided line numbers for syntax errors.

<!-- Created by: Gergő Téringer, 2026 -->