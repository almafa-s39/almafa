# OpenVPN authentication using RADIUS (LDAP)

In this configuration OpenVPN checks the revocation of the clients certificate and uses the CN field as the username. It passes the username to the RADIUS server, which gives back Framed-IP-Address as the ip address of the client.

## Server side

### Install packages for server

```shell
apt install openvpn openvpn-radius wget
```

### Prepare server configuration file

```shell
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn
```

### Edit configuration file: `/etc/openvpn/server.conf`

```cfg
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

verify-client-cert require
plugin /usr/lib/openvpn/radiusplugin.so /etc/openvpn/server/radiusplugin.cnf
username-as-common-name

# Set this if you're resolving issues
verb 6
```

### Copy the radius plugin file and edit the configuration

```shell
cp /usr/share/doc/openvpn-auth-radius/examples/radiusplugin.cnf /etc/openvpn/server/
```

```bash
# /etc/openvpn/server/radiusplugin.cnf
NAS-Identifier=OpenVPN
Service-Type=5
Framed-Protocol=1
NAS-Port-Type=5
NAS-IP-Address=OPENVPN_SERVER_IP
OpenVPNConfig=/etc/openvpn/server.conf

server {
    acctport=1813
    authport=1812
    name=FREERADIUS_SERVER_IP
    retry=1
    wait=1
    sharedsecret=RADIUS_SHARED_SECRET
    requirema=auto
}
```

### Enable and start OpenVPN systemd service

```shell
systemctl enable openvpn@server
systemctl start openvpn@server
```

### [FreeRADIUS setup](/linux/aaa/rad-ldap.md)

## Client side

### Install packages for client

```shell
apt install openvpn openvpn-systemd-resolved resolvconf
```

### Prepare client configuration file

```shell
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn
```

### Edit configuration file: `/etc/openvpn/client.conf`

```shell
# Choose UDP/TCP
remote 193.225.219.17 1194 # Set remote server(s)

# Set certificate settings
ca /ca/ca.crt
cert /ca/user.crt
key /ca/user.key
remote-cert-tls server

auth-user-pass

# Configure up/down scripts
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
```

### Enable and start OpenVPN client systemd service

```shell
systemctl enable openvpn@client
systemctl start openvpn@client
```
