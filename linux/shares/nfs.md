# NFS

## nfs-kernel-serevr

```bash
apt install nfs-kernel-server
```

`/etc/exports`

```bash
<folder>    <access_fqdn/access_subnet>(rw,sync,no_subtree_check)
```

Apply configuration

```bash
exportfs -avz
```

Edit General part in `/etc/idmapd.conf`

```bash
[General]
Verbosity = 0
Pipefs-Directory = /run/rpc_pipefs
Domain = unitel.com
```

Restart service

```bash
systemctl restart nfs-server nfs-idmapd
```

## nfs-client

```bash
# Install service
apt install nfs-common nfs-utils
```

Configure matching idmap settings in `/etc/idmapd.conf`

```bash
[General]
Verbosity = 0
Pipefs-Directory = /run/rpc_pipefs
Domain = unitel.com
```

Configure mount

```bash
# Configure mapping
mkdir /data
echo "<nfs-server>:/<folder>    /data   nfs defaults,_netdev 0 0"
systemctl daemon-reload
mount -a
```
