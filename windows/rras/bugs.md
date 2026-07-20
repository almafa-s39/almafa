# RRAS known bugs and it's solutions

## IKE credentials are unacceptable

Make sure you use the connection with the attributes which are in the SAN fields of the certificate.

If you are trying to create S2S VPN with certificate auth, make sure you went into `Properties > Security` and here you checked in IKE machine certificate.

## A remote access client attempted to connect over a port that was reserved for Routers only

`RRAS > Ports > IKEv2` > Check in: Remote access connections (inbound only), and add more ports as well!
