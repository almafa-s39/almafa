<!-- 
---
title: "roundcube"
author: "Gergő Téringer"
---
 -->
# Roundcube

This document provides administrative procedures for installing and configuring the Roundcube webmail service. It covers package installation, Apache2 and PHP-FPM integration, and Roundcube's IMAP/SMTP connection settings.

## 1. Package Installation and Database Setup

Install the required web server, database, PHP, and Roundcube packages.

```Bash
# Install the core packages
apt install apache2 php-fpm mariadb-server roundcube roundcube-mysql
```

> [!IMPORTANT]
> During the Roundcube installation, you must follow the popup instructions in the terminal for the automatic database configuration.

## 2. Apache2 and PHP-FPM Integration

Enable the necessary Apache2 modules to proxy PHP requests to the PHP-FPM socket.

```Bash
# Enable Apache2 proxy_fcgi and setenvif modules
a2enmod proxy_fcgi setenvif
```

Edit your SSL virtual host configuration file (`/etc/apache2/sites-enabled/default-ssl.conf`) and add the following `FilesMatch` directive inside the `<VirtualHost>` block. This will instruct Apache to pass PHP execution to the PHP 8.2 FPM socket.

```bash
<FilesMatch \.php$>
    SetHandler "proxy:unix:/var/run/php/php8.2-fpm.sock|fcgi://localhost/"
</FilesMatch>
```

Enable the PHP-FPM configuration in Apache2, then restart both services to apply the changes.

```Bash
# Enable the php8.2-fpm config in Apache2
a2enconf php8.2-fpm

# Restart apache2 and php8.2-fpm services
systemctl restart php8.2-fpm apache2

# Create a phpinfo file in the document root for testing purposes
echo '<?php phpinfo(); ?>' > /var/www/html/info.php
```

Verify that PHP is functioning correctly by attempting to access the `info.php` file in your web browser.

## 3. Roundcube Configuration (config.inc.php)

Modify the main Roundcube configuration file (`/etc/roundcube/config.inc.php`) to define your IMAP server, SMTP server, and customize the product name.

```PHP
// Edit the existing variables in /etc/roundcube/config.inc.php
    
// IMAP host chosen to perform the log-in.
// See defaults.inc.php for the option description.
$config['imap_host'] = ["ssl://mail.unitel.com:993"];
    
// SMTP server host (for sending mails).
// See defaults.inc.php for the option description.
$config['smtp_host'] = 'tls://mail.unitel.com:587';
    
// Name your service. This is displayed on the login screen and in the window title.
$config['product_name'] = 'Unitel Webmail Service';
```

Next, append the following extra SMTP and domain parameters to the end of the same configuration file.

```PHP
// EXTRA STUFF appended to the end of the file
$config['smtp_auth_type'] = 'LOGIN';
$config['smtp_helo_host'] = 'mail.unitel.com';
$config['smtp_mail_domain'] = 'unitel.com';
$config['mail_domain'] = 'unitel.com';
```

## 4. Apache Alias and Finalization

Uncomment the alias directive in the Roundcube Apache configuration file to make the webmail interface accessible via the `/roundcube` URL path.

```Bash
# Edit /etc/apache2/conf-enabled/roundcube.conf
# Uncomment line 3:
Alias /roundcube /var/lib/roundcube/public_html
```

```Bash
# Restart the Apache2 service to apply the alias
systemctl restart apache2
```

Finally, try to access the Roundcube interface in your web browser by navigating to `https://<site>/roundcube`.

<!-- Created by: Gergő Téringer, 2026 -->