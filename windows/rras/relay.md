```powershell
netsh routing ip relay install
netsh routing ip relay add dhcpserver <DHCP_Server_IP>
netsh routing ip relay add interface name="<InterfaceName>"
netsh routing ip relay set interface name="<InterfaceName>" relaymode=enable maxhop=4 minsecs=1
```