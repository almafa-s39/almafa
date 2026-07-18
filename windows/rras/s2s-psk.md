# RRAS S2S PSK

## Topology

In this guide, the following topology will be used:

```text
[PARIS-ROUTER] ====== {INTERNET} ====== [LYON-ROUTER]
    20.0.0.2/24                        30.0.0.2/24
```

**FQDNs:**

* paris-router.paris.local
* lyon-router.lyon.paris.local

## RRAS configuration

This guide assumes you already have **RRAS installed and enabled**.

Create a **new demand-dial network interface** with the following settings:

* **Interface name:** Other endpoint's FQDN
* **Select:** *Connect using VPN*
* **Select:** *IKEv2*
* **Host:** Other endpoint's IP address
* **Check:** *Route IP packets*
* Add static routes to reach the other site. On the client side, also add the prefix `192.0.2.0/24`. This will be important later.

Now open **properties** of the interface and in **Options > Connection Type** select *Persistent connection*. Make sure **Security > Authentication** is set to *Use preshared key for authentication* and enter your pre-shared key (PSK).

Alternatively, if you prefer PowerShell, use the following command to create the interface:

```powershell
Add-VpnS2SInterface -Name "<Other FQDN endpoint's>" -AuthenticationMethod PSKOnly -SharedSecret "<YOUR_PRE_SHARED_KEY>" -Persistent -Protocol IKEv2 -IPv4Subnet "<Where ip packets route to>" -Destination "<Other address endpoint's>"
```

The tunnel will need addressing. You have to select a *'server'* and a *'client'* endpoint. In this case, **PARIS-ROUTER** will be the **server** with a **static IP** and an IP pool to give out addresses to other peers.

On the **server** (PARIS-ROUTER), set a **static IP on the interface** in networking. For example, set `192.0.2.1`.

On the **client** (LYON-ROUTER), select *Obtain an IP address automatically* in networking.

Now, **create the pool** on the server: In **RRAS** right-click on `<SERVER_NAME> (local)` and select **Properties**. In **IPv4 > IPv4 Address assignment** select *Static pool* and create pool, e.g. `192.0.2.2 - 192.0.2.100`.

Now, **restart RRAS** on both machines, and the tunnel should connect. To connect the tunnel manually, be sure to connect from the *'client'* peer.

## Verification

To verify that the tunnel is connected and routing traffic correctly, you can use the following PowerShell commands:

```powershell
# Check the status of the Site-to-Site VPN interface 
# The ConnectionState should say 'Connected'
Get-VpnS2SInterface -Name "<Other FQDN endpoint's>"

# View the active Remote Access connection statistics (shows bytes in/out and duration)
Get-RemoteAccessConnectionStatistics

# Verify routing is working by testing connectivity to an IP on the remote subnet
Test-NetConnection -ComputerName <REMOTE_SUBNET_IP>
```
