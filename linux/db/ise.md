<!-- 
---
title: "Setup for Freeradius"
author: "Gergő Téringer"
---
 -->
# Setup for Freeradius

## 1. Install packages and initialize mariadb

```shell
apt install mariadb-server freeradius-mysql
mariadb-secure-install
```

## 2. Create database and insert default Radius scheme into it

> [!NOTE]
> You will need this because

```shell
mysql -u root
```

```sql
CREATE DATABASE radius;
exit;
```

```shell
mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
```

## 3. Create database with only username and IP address

Enter mysql shell

```shell
mysql -u root -p radius
```

Crate table, users, and add privileges for Radius servers

```mysql
# Create your table
CREATE TABLE users (
    username VARCHAR(64) NOT NULL,
    ip       VARCHAR(15) NOT NULL,
    PRIMARY KEY (username),
    UNIQUE KEY uq_username (username)
);

# Fill it with data
INSERT INTO users VALUES ('user1', '10.255.255.101');
INSERT INTO users VALUES ('user2', '10.255.255.102');
INSERT INTO users VALUES ('health', '255.255.255.255'); # Unusable ip for health checks

GRANT ALL ON radius.* TO 'radius'@'10.10.20.%' IDENTIFIED BY 'Passw0rd!";
FLUSH PRIVILEGES:
exit;
```

## 4. Verification

```sql
USE radius;
SELECT * FROM users;
```

<!-- Created by: Gergő Téringer, 2026 -->