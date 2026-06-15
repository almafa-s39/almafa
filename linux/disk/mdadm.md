# mdadm

## Create raid device

Install mdadm

```bash
apt install mdadm
```

List out the attached disks

```bash
lsblk
```

Create the software RAID

```bash
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sd[b-d]
mdam --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

Now you have your RAID block as `/dev/md0`.
