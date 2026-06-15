# LVM

## Physichal volumes
### Create the physichal volume
```bash
pvcreate /dev/md0 
```
### Display the physichal volume
```bash
pvs
pvdisplay /dev/md0 
```
### List all physichal volumes
```bash
pvscan  
```
### Delete phyischal volume
```bash
pvremove /dev/md0 
```
### Resize physichal volumes
```bash
pvresize --setphysicalvolumesize 10G /dev/md0
```

## Volume group
### Create a volume group
```bash
vgcreate storage /dev/md0 # Storage will be the name of the volume group
```
### List volume group
```bash
vgs
vgdisplay storage # You can use vgs by itself also
```
### Rename volume group
```bash
vgrename [old_name] [new_name]
```
### Extend volume groups storage
```bash
vgextend storage [/dev/device_name]
```
### Shrink volume groups storage
```bash
vgreduce storage [/dev/device_name]
```
### Delete volume group
```bash
vgchange -a n storage # Disable volume group
vgremove storage # Delete volume group
```

## Logical volumes
### Create logical volume
```bash
lvcreate -L 2G -n lv_archive storage # Create a logical group into a volume group
```
### Delete logical volume
```bash
lvs
lvdisplay /dev/storage/lv_archive
```
### Rename logical volume
```bash
lvrename [volume_grp_name] [old_name] [new_name]
```
### Extend logical volumes size
```bash
lvextend -L 10G /dev/storage/lv_archive
```
### Shrink logical volumes size
```bash
resize2fs /dev/storage/lv_archive # Extending ext4 filesystem
```
### Delete logical volume
```bash
lvchange -an /dev/vg_dlp/lv_storage # Disable logical volume
lvremove /dev/vg_dlp/lv_storage # Delete logical volume
```

## /etc/fstab
### Create file system on a volume
```bash
mkfs.ext4 /dev/storage/lv_archive
mkfs.ext4 /dev/storage/lv_storage
mkfs.exfat /dev/storage/lv_log
```

### Add the disks to `/etc/fstab/` to mount it every time when device starting
```bash
echo "/dev/storage/lv_archive /archive ext4 defaults 0 2" >> /etc/fstab
echo "/dev/storage/lv_storage /archive ext4 defaults 0 2" >> /etc/fstab
echo "/dev/storage/lv_log /archive exfat defaults 0 2" >> /etc/fstab
```

### Reload the daemon manager, to let the system use the new fstab
```bash
systemctl daemon-reload
mount -a
```
