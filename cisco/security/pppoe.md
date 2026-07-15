# PPPoE

Point-to-Point Protocol over Ethernet (PPPoE) is widely used by Internet Service Providers (ISPs) to deliver authenticated and trackable internet access to end-users over standard Ethernet connections. It combines the session management and authentication features of PPP with the broadcast topology of Ethernet. The configuration requires setting up a Server (usually the ISP router) and a Client (the customer premises router).

## 1. PPPoE Server Configuration

The PPPoE server authenticates the incoming client sessions, establishes a logical Virtual-Template interface for the connection, and assigns an IP address to the client from a local pool.

**Configuration:**

```cisco
username chapuser password 0 Passw0rd!
bba-group pppoe global
 virtual-template 1
exit
interface GigabitEthernet0/3
 no ip address
 pppoe enable group global
 no shutdown
exit
interface Virtual-Template1
 mtu 1492
 ip address 98.76.3.254 255.255.255.0
 peer default ip address pool PPPoE-Pool
 ppp authentication chap
 no shutdown
exit
ip local pool PPPoE-Pool 98.76.3.1
```

**Command Breakdown & Explanation:**

- `username chapuser password 0 Passw0rd!`: Creates a local user database entry. The server will use these credentials to authenticate the client via CHAP. The `0` indicates the password is typed in cleartext.
- `bba-group pppoe global`: Creates a Broadband Access (BBA) group named `global` for PPPoE. This group acts as the bridge between the physical interface and the logical PPP session.
- `virtual-template 1`: Binds the BBA group to `Virtual-Template1`. Every time a client connects, the server clones this template to create a unique Virtual-Access interface for that specific session.
- `interface GigabitEthernet0/3`: The physical interface facing the client. It requires `no ip address` because the IP routing happens on the logical PPP interface, not the physical Ethernet port.
- `pppoe enable group global`: Activates the PPPoE listening service on this interface and associates it with the previously defined BBA group.
- `mtu 1492`: Standard Ethernet MTU is 1500 bytes. PPPoE adds an 8-byte header (6 bytes for PPPoE, 2 bytes for PPP), so the payload MTU must be lowered to `1492` to prevent packet fragmentation.
- `peer default ip address pool PPPoE-Pool`: Instructs the server to assign an IP address to the connecting client using the defined pool.
- `ppp authentication chap`: Enforces Challenge Handshake Authentication Protocol (CHAP), which provides secure, encrypted authentication.
- `ip local pool PPPoE-Pool 98.76.3.1`: Defines the pool of IP addresses to lease to clients. In this specific configuration, it allocates starting from `98.76.3.1`.

**Practical Example:**
When configured on a provider edge (PE) router, this setup allows the router to listen on `GigabitEthernet0/3` for PPPoE discovery broadcasts (PADI). Once a client connects and successfully authenticates as `chapuser`, the server creates a dedicated Virtual-Access interface for them and hands them the IP `98.76.3.1`.

## 2. PPPoE Client Configuration

The PPPoE client initiates the connection by broadcasting discovery packets, authenticating with the server, and dynamically negotiating an IP address for its logical Dialer interface.

**Configuration:**

```cisco
interface Dialer1
 mtu 1492
 ip address negotiated
 encapsulation ppp
 dialer pool 1
 ppp chap hostname chapuser
 ppp chap password Passw0rd!
 no shutdown
 exit
interface GigabitEthernet0/0
 no ip address
 pppoe enable
 pppoe-client dial-pool-number 1
 no shutdown
 exit
```

**Command Breakdown & Explanation:**

- `interface Dialer1`: Creates a logical dialer interface. Unlike a Virtual-Template which is cloned on the server, the Dialer interface permanently represents the client's side of the PPP connection.
- `ip address negotiated`: Tells the client not to use a static IP, but instead request one from the PPPoE server during the IPCP (IP Control Protocol) negotiation phase.
- `encapsulation ppp`: Sets the layer 2 protocol to PPP.
- `dialer pool 1`: Assigns this logical interface to dialer pool `1`. This is how the logical connection maps down to the physical connection.
- `ppp chap hostname` and `ppp chap password`: Provides the credentials that the client will send to the server to prove its identity.
- `interface GigabitEthernet0/0`: The physical interface connecting to the ISP modem or server. Similar to the server side, it needs `no ip address`.
- `pppoe-client dial-pool-number 1`: Binds this physical interface to the logical `Dialer1` interface by referencing the matching pool number `1`.

**Practical Example:**
This is the standard configuration for a branch office router connecting to a DSL or Fiber modem operating in bridge mode. The physical `GigabitEthernet0/0` connects to the modem, but all routing, NAT, and firewall policies should be applied directly to `Dialer1`, as that is where the actual negotiated public IP address and external connection reside.