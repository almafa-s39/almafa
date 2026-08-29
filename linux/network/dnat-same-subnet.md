# Dnat to same subnet

If you want to DNAT on a Debian device into the same subnet, where the device holds an ip address, you have to set the following kernel settings, to make it work:

```bash
sysctl -w net.ipv4.conf.ens192.proxy_arp_pvlan=1
# OR
ip route add <DNAT_ADDRESS>/32 dev <INTER_INTERFACE>
```



```bash
sysctl -w net.ipv4.conf.ens192.proxy_arp=1
# OR
ip neigh add proxy <DNAT_ADDRESS> dev <OUTSIDE_INTERFACE>
```
