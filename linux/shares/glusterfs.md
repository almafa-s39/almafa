<!-- 
---
title: "glusterfs"
author: "Gergő Téringer"
---
 -->
# GlusterFS

This document provides administrative procedures for installing, configuring, and managing a GlusterFS distributed storage cluster on Debian 13 (Trixie). It covers package installation for servers and clients, optional IPv6 transport configuration, peer probing, volume creation (distributed, replicated, and arbiter), and client mounting options including persistent `/etc/fstab` entries.

> [!NOTE]
> GlusterFS is a scalable, network-attached cluster filesystem that aggregates storage building blocks (bricks) from multiple nodes into a unified distributed storage volume.

## 1. Package Installation and Service Initialization

GlusterFS components are split into server management packages for storage nodes and client packages for consumer endpoints.

> [!IMPORTANT]
> Install `glusterfs-server` on all storage nodes participating in the cluster pool. Install `glusterfs-client` on any system that needs to mount the storage volumes.

```Bash
# Install GlusterFS server components on storage nodes
apt install glusterfs-server

# Install GlusterFS client components on client endpoints
apt install glusterfs-client

# Enable and start the GlusterD management service
systemctl enable glusterd --now
```

**Command Breakdown & Explanation:**

- `apt install glusterfs-server`: Installs the GlusterFS server daemon (`glusterd`) and underlying volume management utilities on storage host nodes.
- `apt install glusterfs-client`: Installs FUSE client drivers (`glusterfs`) required to mount GlusterFS volumes on remote endpoints.
- `systemctl enable glusterd --now`: Configures the `glusterd` management service to start at boot and immediately launches the service.

## 2. Optional IPv6 Network Transport Configuration

If operating in an IPv6-enabled environment, the GlusterD service configuration file must be modified to allow IPv6 transport framing.

> [!TIP]
> Modify `/etc/glusterfs/glusterd.vol` on all cluster nodes before probing peers or creating volumes if IPv6 networking is required.

```Bash
# Uncomment or append the IPv6 transport option in /etc/glusterfs/glusterd.vol
# option transport.address-family inet6

# Restart the GlusterD service to apply transport changes
systemctl restart glusterd
```

**Command Breakdown & Explanation:**

- `option transport.address-family inet6`: Directs the `glusterd` management daemon to bind and transmit over IPv6 socket interfaces.
- `systemctl restart glusterd`: Applies configuration updates to the running GlusterD process.

## 3. Trusted Storage Pool Peering

Storage nodes must be linked into a Trusted Storage Pool before volumes can be provisioned.

> [!WARNING]
> Execute the `gluster peer probe` command from **only one** node in the cluster. Do not execute mutual peer probes simultaneously from both nodes.

```Bash
# Probe remote node to establish peer cluster connection (Run on Node 1 only)
gluster peer probe <node_name>
```

**Command Breakdown & Explanation:**

- `gluster peer probe <node_name>`: Sends a peer join request to the specified hostname or IP address, adding it to the trusted storage pool network.

## 4. GlusterFS Volume Creation and Lifecycle Management

GlusterFS supports multiple volume architectures depending on redundancy and performance requirements:

- **Distributed Volume**: Spans files across bricks without replication. Provides maximum capacity and performance but no redundancy.
- **Replicated Volume**: Synchronizes copies of files across multiple bricks for high availability.
- **Distributed & Replicated Volume with Arbiter**: Combines distribution with replication while introducing an arbiter node that stores metadata to prevent split-brain scenarios with lower storage overhead.

```Bash
# 1. Create a Distributed Volume
gluster volume create <volume_name> transport tcp <node1>:<volume_path> <node2>:<volume_path>

# 2. Create a Replicated Volume
gluster volume create <volume_name> replica <node_count> transport tcp <node1>:<volume_path> <node2>:<volume_path>

# 3. Create a Distributed & Replicated Volume with Arbiter
gluster volume create <volume_name> replica <replica_node_count> arbiter <arbiter_node_count> transport tcp <node1>:<volume_path> <node2>:<volume_path> <arbiter_node>:<volume_path>

# Start the newly provisioned volume
gluster volume start <volume_name>
```

**Command Breakdown & Explanation:**

- `gluster volume create`: Initializes a logical GlusterFS volume across specified bricks (`node:path`).
- `replica <node_count>`: Specifies the number of synchronous file copies maintained across distinct nodes.
- `arbiter <arbiter_node_count>`: Designates specific brick endpoints to store file metadata/checksums rather than full data payloads to resolve split-brain conflicts economically.
- `gluster volume start <volume_name>`: Activates the volume, enabling client connection endpoints.

## 5. Client Volume Mounting and Automounting

GlusterFS volumes are mounted on clients using the native `glusterfs` FUSE helper.

### 5.1 Manual Client Mounting

```Bash
# Manually mount the GlusterFS volume on the client endpoint
mount -t glusterfs <node_name>:/<volume_name> <mount_point>
```

**Command Breakdown & Explanation:**

- `mount -t glusterfs`: Mounts the network volume using the GlusterFS FUSE client wrapper, targeting any active cluster storage node identifier (`<node_name>:/<volume_name>`).

### 5.2 Persistent Mount Configuration via fstab

To automatically mount GlusterFS volumes at system boot, add an entry to `/etc/fstab`.

> [!IMPORTANT]
> Always include the `_netdev` mount option in `/etc/fstab` to ensure the operating system delays mounting the volume until network interfaces are fully initialized during boot.

```Plaintext
# Standard IPv4 /etc/fstab entry:
<node_name>:/<volume_name> <mount_point> glusterfs defaults,_netdev 0 0

# IPv6-enabled /etc/fstab entry:
<node_name>:/<volume_name> <mount_point> glusterfs defaults,_netdev,xlator-option=transport.address-family=inet6 0 0
```

**Command Breakdown & Explanation:**

- `defaults,_netdev`: Ensures standard mount permissions while enforcing network initialization dependency before mounting.
- `xlator-option=transport.address-family=inet6`: Directs the client FUSE translator to establish the mount socket over IPv6 transport.

## 6. Verification and Troubleshooting

> [!NOTE]
> Validate cluster peering status, volume states, brick health, and active client mounts on Debian 13 using standard administrative commands.

### 6.1 Verify GlusterD service running status

**Command:** `systemctl status glusterd`

**What it checks and variables to look for:**

- **Active**: Must be `active (running)`
- **Loaded**: Must be `loaded (/usr/lib/systemd/system/glusterd.service; enabled)`

### 6.2 Verify peer status and trusted storage pool connectivity

**Command:** `gluster peer status`

**What it checks and variables to look for:**

- **State**: Must be `Peer in Cluster (Connected)`
- **Number of Peers**: Must reflect expected peer count (e.g., `1` or higher)

### 6.3 Verify GlusterFS volume health and operational parameters

**Command:** `gluster volume info`

**What it checks and variables to look for:**

- **Status**: Must be `Started`
- **Type**: Must display `Replicate`, `Distribute`, or `Distributed-Replicate`
- **Transport-type**: Must be `tcp`

### 6.4 Verify client volume mount status and filesystem type

**Command:** `df -hT <mount_point>`

**What it checks and variables to look for:**

- **Type**: Must display `fuse.glusterfs` or `glusterfs`
- **Mounted on**: Must match target `<mount_point>` directory

<!-- Created by: Gergő Téringer, 2026 -->