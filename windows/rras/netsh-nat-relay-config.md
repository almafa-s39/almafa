# Configure NAT and DHCP Relay from powershell ( or cmd )

First you have to have an installed RRAS server, for at least routing.

NAT config:
```powershell
# Install new routing protocol NAT
netsh routing ip nat install

# Assig inside, outside interfaces - with this one PAT will automatically work
netsh routing ip nat add interface <INTERFACE> mode=full # Public interface
netsh routing ip nat add interface <INTERFACE> mode=private # Private interface

# Create portforwarding or as windows calls portmapping
netsh routing ip nat add portmapping <PROTO> <INTERNET-LISTEN=ADDRESS> <INTERNET-PORT> <PRIVATE-ADDRESS> <PRIVATE-PORT>
netsh routing ip nat add portmapping tcp 0.0.0.0 80 10.10.10.10 80
```

If additionally somewhy you want to do more, so map to another address the port forwarding, you can do it by the following steps:
```powershell
# Add a secondary address to your interface
netsh int ipv4 add add Ethernet0 203.0.113.11 255.255.255.0

# Create the nat pool in RRAS
netsh routing ip nat add addressrange Ethernet0 203.0.113.11 203.0.113.11 255.255.255.0

# Create a portforward to that interface
netsh routing ip nat add portmapping tcp 203.0.113.11 443 10.10.10.10 443
```

Verify it with these commands:

```powershell
netsh routing ip nat show interface
```


DHCP Relay config:
```powershell
# Install new routing protocol DHCP Relay
netsh routing ip relay install

# Add DHCP Server to forward messages
netsh routing ip relay add dhcpserver <SERVER'S ADDRESS>

# Add interfaces to dhcp relay
netsh routing ip relay add interface <INTERFACE>
```


Verify it with these commands:

```powershell
netsh routing ip relay show interface
```
