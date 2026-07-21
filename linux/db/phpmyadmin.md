<!-- 
---
title: "phpMyAdmin and Apache2 Web Server Installation Guide"
author: "Gergő Téringer"
---
 -->
# phpMyAdmin and Apache2 Web Server Installation Guide

This document provides administrative procedures for deploying the Apache2 web server alongside phpMyAdmin on Debian 13 (Trixie). phpMyAdmin is a free software tool written in PHP, intended to handle the administration of MySQL or MariaDB over the Web.

> [!NOTE]
> It is assumed that a local or remote MySQL/MariaDB database server is already installed and running before starting this deployment.

## 1. Apache2 and PHP Dependencies Installation

phpMyAdmin requires a web server to serve its interface and PHP extensions to communicate with the database and process web elements.

> [!IMPORTANT]
> Debian 13 utilizes PHP 8.2 (or newer) by default. Installing the core `php` meta-package ensures the correct default version is pulled from the repositories along with the required Apache2 PHP module.

```bash
# Update repositories and install Apache2 along with required PHP extensions
apt install apache2 php libapache2-mod-php php-mysql php-mbstring php-zip php-gd php-json php-curl php-xml

# Enable and start the Apache2 service
systemctl enable apache2 --now
```

**Command Breakdown & Explanation:**

- `apache2`: The core Apache HTTP web server daemon.
- `php libapache2-mod-php`: Installs the PHP scripting language and the module required for Apache2 to process PHP files.
- `php-mysql php-mbstring php-zip...`: Installs mandatory and highly recommended PHP extensions required by phpMyAdmin for database communication, string manipulation, and archive extraction.
- `systemctl enable apache2 --now`: Configures the Apache2 service to start on boot and starts it immediately.

## 2. phpMyAdmin Package Installation

Debian provides a pre-packaged version of phpMyAdmin that automatically configures the Apache2 web server alias and assists with the initial database setup using `dbconfig-common`.

> [!WARNING]
> During the installation, an interactive prompt will appear asking to choose the web server that should be automatically configured. You **must** press the Spacebar to select `apache2` (an asterisk `*` will appear) before pressing Enter.

```bash
# Install the phpMyAdmin package
apt install phpmyadmin
```

**Command Breakdown & Explanation:**

- `apt install phpmyadmin`: Downloads and extracts the phpMyAdmin web application. During this process:
  1. Select **apache2** when prompted for the web server.
  2. Select **Yes** when asked to use `dbconfig-common` to set up the database.
  3. Provide a MySQL/MariaDB application password for phpMyAdmin to register itself within the database server.

## 3. Apache2 Configuration and Security Hardening

The Debian package automatically places an Apache configuration file at `/etc/apache2/conf-available/phpmyadmin.conf` and creates an alias mapping `/phpmyadmin` to `/usr/share/phpmyadmin`.

> [!TIP]
> Changing the default `/phpmyadmin` URL alias to a customized, hidden path (e.g., `/db-admin-portal`) mitigates automated bot scanners attempting to brute-force the default login page.

```bash
# Explicitly enable the phpMyAdmin configuration in Apache (usually done automatically by apt)
a2enconf phpmyadmin

# Enable the rewrite and php modules (if not already enabled)
a2enmod rewrite

# Optional: Change the default alias URL to obscure the login page
sed -i 's|Alias /phpmyadmin|Alias /secure-db-login|' /etc/apache2/conf-available/phpmyadmin.conf

# Reload Apache2 to apply configuration changes
systemctl reload apache2
```

**Command Breakdown & Explanation:**

- `a2enconf phpmyadmin`: Creates a symlink from `conf-available` to `conf-enabled`, instructing Apache to load the phpMyAdmin configuration block.
- `a2enmod rewrite`: Enables the Apache URL rewrite module, often required for modern web frameworks and routing.
- `sed -i 's|Alias /phpmyadmin|Alias /secure-db-login|' ...`: Replaces the default URL path so that the interface is accessed via `http://server-ip/secure-db-login` instead of the easily guessable `/phpmyadmin`.
- `systemctl reload apache2`: Gracefully reloads the Apache daemon to apply new configurations without dropping active HTTP connections.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the web server status, port bindings, and HTTP response codes on Debian 13 using standard administrative tools to ensure phpMyAdmin is accessible.

### 4.1 Verify Apache2 service running status

**Command:** `systemctl status apache2`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/apache2.service; enabled)`

### 4.2 Verify network listening state on TCP port 80 (HTTP)

**Command:** `ss -tuln | grep :80`

**What it checks and variables to look for:**

- **State**: Must be `LISTEN`
- **Local Address:Port**: Must display `*:80` or `0.0.0.0:80`

### 4.3 Verify phpMyAdmin Apache configuration syntax

**Command:** `apache2ctl configtest`

**What it checks and variables to look for:**

- **Syntax output**: Must return `Syntax OK`

### 4.4 Verify HTTP access to the phpMyAdmin alias

**Command:** `curl -I http://127.0.0.1/phpmyadmin/` *(or use your custom alias if changed)*

**What it checks and variables to look for:**

- **HTTP Status Code**: Must be `HTTP/1.1 200 OK`
- **Content-Type**: Must be `text/html`

<!-- Created by: Gergő Téringer, 2026 -->