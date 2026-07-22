<!-- 
---
title: "Cacti Installation and Setup"
author: "Gergő Téringer"
---
 -->

# Cacti Installation and Setup

## 1. Install required packages

Install the necessary packages:

```bash
apt install cacti snmp snmpd php-mysql php-snmp php-intl rrdtool
```

When prompted, choose Apache2 and do not create a database.

## 2. Configure MariaDB

Edit the MariaDB configuration file:

```bash
/etc/mysql/mariadb.conf.d/50-server.cnf
```

If these values are not set, the web GUI may show errors.

```ini
[mariadbd]
collation-server=utf8mb4_unicode_ci
max_heap_table_size=128M
tmp_table_size=128M
innodb_doublewrite=OFF
innodb_buffer_pool_size=2048M
```

Create the database:

```bash
mysql -u root -p
```

```sql
create database cacti;
grant all privileges on cacti.* to cacti@'localhost' identified by 'password';
grant select on mysql.time_zone_name to cacti@'localhost';
exit;
```

Run the following commands:

```bash
mariadb-tzinfo-to-sql /usr/share/zoneinfo | mysql -u root mysql
mysql -u cacti -p cacti < /usr/share/doc/cacti/cacti.sql
```

## 3. Configure Cacti PHP settings

Edit the PHP configuration file:

```bash
/usr/share/cacti/site/include/config.php
```

```php
$database_type     = "mysql";
$database_default  = "cacti";
$database_hostname = "127.0.0.1";
$database_username = "cacti";
$database_password = "password";
$database_port     = "3306";
$database_retries  = 5;
$database_ssl      = false;
```

## 4. Configure Apache2

Edit the Apache configuration file if you want to restrict which IP addresses can access the Cacti site:

```bash
/etc/apache2/conf-available/cacti.conf
```

## 5. Fix file ownership

Change the ownership of the required folders and files:

```bash
chown -R www-data:www-data /usr/share/cacti/resource
chown -R www-data:www-data /usr/share/cacti/site/scripts
chown www-data:www-data /var/log/cacti/poller-error.log
chown www-data:www-data /var/log/cacti/rrd.log
```

## 6. Restart services

Restart Apache2 and MariaDB:

```bash
systemctl reload apache2
```

## 7. Access the web interface

Open the Cacti web interface:

```text
http://<IP or DNS>/cacti
```

Default login credentials:

- Username: admin
- Password: admin

Complete the installation steps in the web interface.

## 8. Add and organize devices

### 8.1 Monitor a new device

1. Go to Create > New device.
2. Follow the instructions.
3. Verify the device in Management > Devices.

### 8.2 Create a new tree

1. Go to Management > Trees.
2. Click the plus button to create a new tree.
3. You can use the default tree if preferred.
4. Drag devices into the tree items.

### 8.3 Add graphs to the tree

1. Go to Management > Graphs.
2. Select the graphs you want to add.
3. Choose Place on a Tree and select the tree name.
4. Click Go.

Your tree will appear in the Graphs tab once it is published. Creating graphs may take some time.

### 8.4 Group devices by site

You can organize devices into sites by creating a new site under Management > Sites and then selecting that site in the device settings under Management > Devices.

<!-- Created by: Gergő Téringer, 2026 -->