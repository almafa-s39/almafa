<!-- 
---
title: "TS - Else"
author: "Gergő Téringer"
---
-->
# TS - Else

> [!NOTE]
> Here you can find everything that has not been applied to any other modules

## No firewall options configured (NFTables, IPtables)

Check `/etc/{hosts.allow,hosts.deny}`

## vCenter Template 95% stuck and 30 min

> [!NOTE]
> This issue came out when using vCenter version 8.0.2.0000, in 2b it is fixed.

The issue comes from because the users password is expired. First of all change your password, but the faster option is that you SSH in to your vCenter and issue the following command: `service-control --restart vsm`.

<!-- Created by: Gergő Téringer, 2026 -->