<!-- 
---
title: "lsyncd"
author: "Gergő Téringer"
---
 -->

# One-way file syncing

## 1. Introduction

This guide will demonstrate the configuration of **lsyncd** in order to achieve a **one-way sync** of a folder between **two hosts**.

## 2. Installation

Install lsyncd with the following command:

```bash
apt install lsyncd
```

## 3. Configuration

To configure lsyncd with rsync ssh copy, copy the example file to `/etc`.

```bash
mkdir /etc/lsyncd
cp /usr/share/doc/lsyncd/examples/lrsyncssh.lua /etc/lsyncd/lsyncd.conf.lua
```

Now, edit the variables you need in the file:

```lua
sync { 
  default.rsyncssh,
  source="/path/to/source",
  host="<destination-host>",
  targetdir="/path/to/destination"
}
```

If you need more source/destination pairs, you can copy the sync block.

> [!WARNING]
> Be aware that the source and destination can only be **directories, not individual files.**

Because rsync uses ssh to copy files in the background, you need to set up passwordless ssh. In order to do so, enter the following commands on the source computer:

```bash
ssh-keygen
ssh-copy-id <destination-ip>
```

Now, restart and enable lsyncd:

```bash
service lsyncd restart
systemctl enable lsyncd
```

<!-- Created by: Gergő Téringer, 2026 -->