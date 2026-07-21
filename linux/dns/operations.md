<!-- 
---
title: "Bind9 useful commands"
author: "Gergő Téringer"
---
 -->
# Bind9 useful commands

## Zone update

If you have a zone file in raw format, you can convert it back into text format with the following command:

```shell
nsupdate -k /etc/bind/update.key <<EOF
server <DNS_SERVER>
zone "<DOMAIN_NAME>"
update {add} <RECORD>.<DOMAIN_NAME>. 86400 <TYPE> <TARGET>
/
update {delete} <RECORD>.<DOMAIN_NAME>. <TYPE>
show
send
EOF
```

## Check transfered zone

If you have a zone file in raw format, you can convert it back into text format with the following command:

```shell
named-compilezone -f raw -F text -o output.txt domain.com zonefile.raw
```

<!-- Created by: Gergő Téringer, 2026 -->