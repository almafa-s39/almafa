# BGP

## Install RRAS

```powershell
Install-WindowsFeature -Name Routing -IncludeManagementTools
Restart-Copmuter # It will promt you to restart your computer to finish installing


Install-RemoteAccess -VpnType RoutingOnly
```

## Configure BGP

```powershell
Add-BgpRouter -BgpIdentifier <RID> -LocalASN <LOCAL_AS> 
Set-BgpRouter -BgpIdentifier <RID> -IPv6Routing Enabled

Add-BgpPeer -PeerIPAddress <PEER_ADDRESS> -LocalIPAddress <LOCAL_PEER_ADDRESS> -PeerASN <PEER_AS> -PeerName <STRING> -LocalASN <LOCAL_AS>
Add-BGPCustomRoute -Network <NETWORK_ADD>/<NETMASK>
```
