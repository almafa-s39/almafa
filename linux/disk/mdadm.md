<!-- 
---
title: "mdadm"
author: "Gergő Téringer"
---
-->
# mdadm

## Create raid device

Install mdadm

```shell
apt install mdadm
```

List out the attached disks

```shell
lsblk
```

Create the software RAID

```shell
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sd[b-d]
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

Now you have your RAID block as `/dev/md0`.

<!-- Created by: Gergő Téringer, 2026 -->