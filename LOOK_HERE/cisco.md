# PPPoE

Minden ami kell benne van a leírásban.

[link](/cisco/services/pppoe.md)

Az ip local pool legyen 1 hosszú a feladathoz.

Teljes konfig minta:

```cisco
interface Loopback0
 ip address 192.168.100.1 255.255.255.0
!
ip local pool PPPOE-POOL 192.168.100.10 192.168.100.100
!
username client1 password 0 CHANGE-ME-STRONG-PASSWORD
!
bba-group pppoe PPPOE-GROUP
 virtual-template 1
!
interface Virtual-Template1
 ip unnumbered Loopback0
 peer default ip address pool PPPOE-POOL
 ppp authentication chap
 ppp mtu adaptive
!
interface GigabitEthernet0/1
 description WAN-facing interface toward PPPoE clients
 no ip address
 pppoe enable group PPPOE-GROUP
 no shutdown
```

Teljes kliens minta:

```cisco
interface GigabitEthernet0/0
 description WAN link to ISP / PPPoE Access Concentrator
 no ip address
 pppoe enable group global
 pppoe-client dial-pool-number 1
 no shutdown
!
interface Dialer1
 mtu 1492
 ip address negotiated
 ip nat outside
 encapsulation ppp
 dialer pool 1
 dialer-group 1
 ppp authentication chap callin
 ppp chap hostname client1
 ppp chap password 0 CHANGE-ME-STRONG-PASSWORD
!
dialer-list 1 protocol ip permit
!
ip route 0.0.0.0 0.0.0.0 Dialer1
```
