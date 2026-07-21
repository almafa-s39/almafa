<!-- 
---
title: "Quota"
author: "Gergő Téringer"
---
-->
# Quota

```bash
# Install packages
apt install quota

# Add internal quota
umount /dev/md0
tune2fs -O quota /dev/md0

# Add these settings to your  disk where you want to apply quota, in the `/etc/fstab/` file
/dev/sda1       /home   ext4 defaults,usrquota,grpquota 0 2

# Reload daemon and mount it again with the new settings
systemctl deamon-reload
mount -a

# Create quota using the following command
setquota -u <username> <soft_limit> <hard_limit> 0 0 <path>
setquota -t <block_grace_time> <inode_grace_time> <path>

# List datas
quota -vs <username>
repquota -s <path> | grep "Block"
```

<!-- Created by: Gergő Téringer, 2026 -->