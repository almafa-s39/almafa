# PBR(iproute2)

Install `iproute2` package using apt.

```bash
apt install iproute2
```

Create a new "VRF"

```bash
echo "100 research" >> /etc/iproute2/rt_tables
```

Create a new route in the route table and then add a new PBR

```bash
ip route add 10.10.20.0/24 dev ipsec0 via 10.255.255.2 table research
ip rule add from 10.10.10.0/24 lookup research prio 100
```

Create a script into /etc/bashrc and give it sufficient permissions to run at startup, to add everytime the server starts create the new route
