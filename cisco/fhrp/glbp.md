# GLBP

GLBP load balances between routers, so there is more than a HA like in HSRP. It works with IPv6 as well! It uses nearly the same command, you just have to swap standby and GLBP.

## Interface level configurations

```
interface Vlan103
 ip address 10.10.103.2 255.255.255.0
 glbp 103 ip 10.10.103.1
 glbp 103 priority 110
 glbp 103 preempt [ delay minimum 10 ]
 glbp 103 load-balancing host-dependent
 glbp 103 authentication md5 key-string Passw0rd!
 glbp 103 name MGMT
 glbp 103 weighting track 10 decrement 20
 glbp 103 forwarder preempt [ delay minimum 10 ]
end
```
