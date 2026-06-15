# OpenVPN LowBudget ISE (MySQL)

In this configuration OpenVPN checks the revocation of the clients certificate and uses the CN field as the username. It passes the username to the RADIUS server, which gives back Framed-IP-Address as the ip address of the client.

## Server side

### Install packages
```bash
apt install openvpn freeradius-utils wget
```

### Prepare configuration file
```bash
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn
```

### Edit configuration file: `/etc/openvpn/server.conf`
```conf
# Choose UDP/TCP.
# Edit CA settings (you need serverAuth extension on the certificate and the full chain as CA)
ca /ca/chain.pem
cert /ca/server.crt
key /ca/server.key
dh /ca/dh.pem # Issue the command above

server 10.255.255.0 255.255.255.0 # Use your subnet
push "route 10.10.0.0 255.255.0.0" # Push route to clients
push "dhcp-option DNS 10.10.10.100" # Push DNS
push "dhcp-option DOMAIN-ROUTE ."

crl-verify /etc/openvpn/ca.crl
remote-cert-tls client
verify-client-cert require

script-security 2
client-connect /etc/openvpn/radius_auth.sh
```

### `/etc/openvpn/radius_auth.sh` (chmod +x)
```bash
#!/bin/bash

RADIUS_SERVER="10.10.10.100"
RADIUS_SECERT="Passw0rd!"
RESPONSE=$(echo "User-Name=$common_name, User-Passw0rd=certauth" | radclient -x $RADIUS_SERVER:1812 auth $RADIUS_SECRET 2>/dev/null )

if echo "$RESPONSE" | grep -q "Access-Accept"; then
    FRAMED_IP=$(echo "$RESPONSE" | grep "Framed-IP-Address" | awk '{print $3}' | tr -d "'")
    if [ -n "$FRAMED_IP" ]; then
        echo "ifconfig-push $FRAMED_IP 255.255.255.0" > "$1"
    fi
    exit 0
else
    exit 1
fi
```

### Download crl 
Enter `crontab -e` and add the following line to the end:

```bash
*/10 * * * * * wget -O /etc/openvpn/ca.crl http://pki.company.com/ca.crl
```

### Enable and start OpenVPN systemd service
```bash
systemctl enable openvpn@server
systemctl start openvpn@server
```


### [FreeRADIUS setup](/linux/aaa/rad-sql.md)

## Client side

### Install packages
```bash
apt install openvpn openvpn-systemd-resolved
```

### Prepare configuration file
```bash
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn
```

### Prepare scripts for the configuration
```bash
echo "cp /etc/openvpn/resolv.conf.up /etc/resolv.conf" > /etc/openvpn/down.sh 
echo "cp /etc/openvpn/resolv.conf.down /etc/resolv.conf" > /etc/openvpn/down.sh
chmod +x /etc/openvpn/*.sh
echo "nameserver 10.10.10.100" > /etc/openvpn/resolv.conf.up
echo -e "nameserver 127.0.0.53\noptions edns0 trust-ad\nsearch ." > /etc/openvpn/resolv.conf.down
```

### Edit configuration file: `/etc/openvpn/client.conf`
```bash
# Choose UDP/TCP
remote 193.225.219.17 1194 # Set remote server(s)

# Set certificate settings
ca /ca/ca.crt
cert /ca/user.crt
key /ca/user.key
remote-cert-tls server

# Configure up/down scripts
script-security 2
up /etc/openvpn/up.sh
down /etc/openvpn/down.sh
```

### Enable and start OpenVPN systemd service
```bash
systemctl enable openvpn@client
systemctl start openvpn@client
```