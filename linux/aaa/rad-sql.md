# RADIUS SQL Settings (LowBudget ISE)

## Install packages

```shell
apt install freeradius freeradius-mysql freeradius-utils
cd /etc/freeradius/3.0/
```

## Disable default site named "inner-tunnel"
```shell
rm sites-enabled/inner-tunnel
```

## Edit your default site configuration
Edit `sites-enabled/default`, and replace the authorization block to this:
```shell
authorize {
    preprocess


    # Look up the certificate CN (sent as User-Name) in the vpn.users table
    update control {
        &Tmp-String-0 := "%{sql:SELECT ip FROM users WHERE username = '%{User-Name}'}"
    }

    if (&control:Tmp-String-0 && &control:Tmp-String-0 != "") {
        update reply {
            &Framed-IP-Address := "%{control:Tmp-String-0}"
        }
        update control {
            &Auth-Type := Accept
        }
    }
    else {
        reject
    }
}
```

## Add your client with secret
Edit `./clients.conf` and add to the top the following part:
```shell
client openvpn_server {
	ipaddr = 10.10.10.254
	secret = Passw0rd!
    shortname = ovpn
}
```

## Edit mysql module configuration
Edit `mods-available/sql` file. You have to edit inside the sql { ... } block. There will be a lot of comments inside the commands!
```shell
sql {
    dialect = "mysql"
    driver = "rlm_sql_${dialect}"
    # You have to add this part by hand ====
    server = 10.10.20.10
	port = 3306
	login = "radius" # username
	password = "Passw0rd!"
	radius_db = "vpn"
    read_client = no
    # =====================================

    mysql { 
        tls {
            # Comment out everything here if you will not set up TLS for YourSQL :)
        }
    }
}
```


## Enable mysql mod
Create a symbolic link form sql mod (use full path):
```shell
ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/sql
```

## Restart the service

```shell
systemctl restart freeradius
```


## Last step:
### [OpenVPN setup](/linux/vpn/ovpn-ise.md)

## Next step:
### [Database setup](/linux/db/ise.md)