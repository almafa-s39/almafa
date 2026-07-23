<!-- 
---
title: "Rsync"
author: "Gergő Téringer"
---
 -->
# Rsync

This document covers installing and using `rsync` on Debian 13.3
(Trixie), including local synchronization, remote synchronization over
SSH, include/exclude filtering, automation, and running `rsync` as a
standalone daemon (`rsyncd`) for non-SSH transfers.

Rsync synchronizes files and directories efficiently by transferring
only the parts of files that have changed (the "rsync algorithm"),
rather than re-copying entire files on every run. It can operate in
three modes: local (both paths on the same host), remote-shell (over
SSH), and daemon (a standalone `rsync://` service, typically on TCP
port 873).

> [!NOTE]
> On Debian 13, the single `rsync` package provides both the `rsync`
> command-line client and the optional `rsync` daemon functionality —
> there is no separate `rsync-daemon` package as on some other
> distributions.

## 1. Installation

Install the package from the standard Debian repositories.

```bash
apt install rsync
```

**Command Breakdown & Explanation:**

- `apt install rsync`: Installs the `rsync` binary, man pages, and the
  daemon-related systemd unit (`rsync.service`), which remains disabled
  by default until you configure and enable it (see Section 8).

```bash
rsync --version
```

Confirms the installed version and compiled-in capabilities (e.g.,
`xxhash`, `zstd`, `lz4` compression support), which is useful when
troubleshooting compatibility with older `rsync` versions on remote
hosts.

## 2. Basic Local Synchronization

The most common invocation combines several flags into `-avz` or
`-avzP`. A trailing slash on the source path matters: it copies the
*contents* of the source directory into the destination, rather than
the directory itself.

```bash
rsync -avzP /home/user/documents/ /mnt/backup/documents/
```

**Command Breakdown & Explanation:**

- `-a` (`--archive`): Preserves permissions, ownership, timestamps,
  symlinks, and recurses into subdirectories. This is the standard flag
  for backup-style copies.
- `-v` (`--verbose`): Lists each file as it is transferred.
- `-z` (`--compress`): Compresses file data during transfer. Most useful
  over slow or remote links; provides little benefit on a fast local
  copy.
- `-P`: Shorthand for `--progress --partial`. Shows a per-file progress
  bar and keeps partially transferred files if the transfer is
  interrupted, so a re-run can resume rather than restart.
- `/home/user/documents/` (trailing slash): Copies the contents of
  `documents/` into the destination. Omitting the trailing slash would
  instead create a `documents/` subdirectory inside the destination.

> [!TIP]
> Always test a new `rsync` command with `--dry-run` (or `-n`) first,
> especially when combined with `--delete`. It prints exactly what would
> happen without touching any files.

## 3. Remote Synchronization over SSH

For transfers between hosts, `rsync` defaults to using SSH as the
remote-shell transport, so no separate daemon or open port is required
on the remote host beyond SSH itself (TCP 22). SSH key-based
authentication is strongly recommended over passwords for unattended
transfers.

### 3.1 Push (Local to Remote)

```bash
rsync -avzP -e ssh /home/user/documents/ deploy@203.0.113.10:/srv/backup/documents/
```

**Command Breakdown & Explanation:**

- `-e ssh`: Explicitly selects SSH as the remote shell. This is
  `rsync`'s default when a remote host is detected in the path, so `-e
  ssh` is often omitted; it is shown here for clarity and because it is
  required when passing extra SSH options (see Section 3.3).
- `deploy@203.0.113.10:/srv/backup/documents/`: The remote destination,
  in `user@host:path` form. `rsync` connects over SSH as `deploy` and
  writes into `/srv/backup/documents/` on the remote host.

### 3.2 Pull (Remote to Local)

```bash
rsync -avzP deploy@203.0.113.10:/srv/backup/documents/ /home/user/documents/
```

**Command Breakdown & Explanation:**

- Reversing the source and destination arguments pulls files from the
  remote host down to the local machine instead of pushing them. All
  other flags behave identically.

### 3.3 Using a Non-Default SSH Port or Key

```bash
rsync -avzP -e "ssh -p 2222 -i /home/user/.ssh/id_ed25519_backup" /home/user/documents/ deploy@203.0.113.10:/srv/backup/documents/
```

**Command Breakdown & Explanation:**

- `-e "ssh -p 2222 -i ..."`: Passes extra arguments directly to the
  underlying `ssh` command. Use this when the remote SSH daemon listens
  on a non-standard port or when a specific private key must be used
  rather than the default `~/.ssh/id_rsa` / `~/.ssh/id_ed25519`.

> [!IMPORTANT]
> For unattended or scripted transfers (cron jobs, systemd timers),
> configure passwordless SSH key authentication and, ideally, restrict
> the key with a `command=` forced-command entry in the remote
> `authorized_keys` file (or use `rrsync`, included with the `rsync`
> package) so the key can only run `rsync`, not an arbitrary shell.

## 4. Common Options Reference

The table below summarizes frequently used flags beyond the `-avzP`
baseline.

| Option | Effect |
| --- | --- |
| `--delete` | Deletes files in the destination that no longer exist in the source, keeping the destination an exact mirror. |
| `-n`, `--dry-run` | Shows what would be transferred/deleted without making any changes. |
| `--checksum` | Compares files by checksum instead of size and modification time; slower but catches changes that don't alter the timestamp. |
| `--bwlimit=RATE` | Caps transfer bandwidth (e.g., `--bwlimit=5m` for 5 MB/s), useful on shared or metered links. |
| `--exclude=PATTERN` | Skips files/directories matching `PATTERN` (see Section 5). |
| `--link-dest=DIR` | Hard-links unchanged files from `DIR` instead of re-copying them, enabling space-efficient incremental snapshot backups. |
| `-u`, `--update` | Skips files that are newer on the destination than in the source. |
| `--stats` | Prints a summary of bytes transferred, speedup ratio, and file counts at the end of the run. |

> [!WARNING]
> `--delete` is destructive. Always pair it with `--dry-run` on first
> use, and double-check the direction of the sync — running it in the
> wrong direction can permanently remove files from what you intended to
> be the "good" copy.

## 5. Include and Exclude Patterns

Exclude and include patterns control which files are transferred.
Patterns can be given inline or read from a file.

```bash
rsync -avzP --exclude='*.tmp' --exclude='.cache/' /home/user/project/ /mnt/backup/project/
```

```bash
rsync -avzP --exclude-from=/etc/rsync_exclude.lst /home/user/project/ /mnt/backup/project/
```

**Command Breakdown & Explanation:**

- `--exclude='*.tmp'`: Skips any file matching the glob pattern
  `*.tmp`, at any depth.
- `--exclude='.cache/'`: The trailing slash restricts the match to
  directories named `.cache`, leaving any regular file named `.cache`
  untouched.
- `--exclude-from=/etc/rsync_exclude.lst`: Reads one pattern per line
  from the given file, useful for long or reusable exclude lists shared
  across multiple `rsync` invocations.

> [!NOTE]
> Rule order matters: `rsync` evaluates `--include`/`--exclude` rules in
> the order given, and the first matching rule wins. To exclude
> everything except a specific subdirectory, an explicit `--include` for
> that subdirectory must come before a catch-all `--exclude='*'`.

## 6. Automating Rsync (Cron and Systemd Timers)

Recurring transfers are typically scheduled with either a cron job or a
systemd timer. A systemd timer is generally preferred on Debian 13, as
it integrates with `journalctl` logging and handles missed runs
(`Persistent=true`) more gracefully than cron.

```bash
# /etc/cron.d/rsync-backup
15 2 * * * deploy rsync -az --delete /home/user/documents/ deploy@203.0.113.10:/srv/backup/documents/ >> /var/log/rsync-backup.log 2>&1
```

```bash
# /etc/systemd/system/rsync-backup.service
[Unit]
Description=Rsync backup to remote host

[Service]
Type=oneshot
User=deploy
ExecStart=/usr/bin/rsync -az --delete /home/user/documents/ deploy@203.0.113.10:/srv/backup/documents/
```

```bash
# /etc/systemd/system/rsync-backup.timer
[Unit]
Description=Run rsync-backup daily

[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Command Breakdown & Explanation:**

- `15 2 * * *` (cron): Runs daily at 02:15.
- `Type=oneshot`: Tells systemd this service runs to completion and
  exits, rather than staying resident.
- `OnCalendar=*-*-* 02:15:00`: Systemd calendar expression equivalent to
  the cron schedule above.
- `Persistent=true`: If the system was powered off at the scheduled
  time, the job runs as soon as the system is back up, rather than
  waiting for the next scheduled occurrence.

## 7. Rsync Daemon Mode

Daemon mode (`rsync://host/module` syntax, or `host::module`) runs
`rsync` as a standalone service listening on TCP port 873, without
requiring SSH on the target host. It is commonly used for anonymous or
semi-trusted read-only mirrors, or environments where SSH access is
undesirable. Because daemon-mode traffic is unencrypted by default,
restrict it to trusted networks or tunnel it over SSH/VPN if
confidentiality matters.

### 7.1 Daemon Configuration (rsyncd.conf)

Debian does not ship a default `/etc/rsyncd.conf`; create it from
scratch. The file defines global settings followed by one or more
named modules.

```bash
# /etc/rsyncd.conf
uid = nobody
gid = nogroup
use chroot = yes
max connections = 4
pid file = /run/rsyncd.pid
log file = /var/log/rsyncd.log

[backup]
path = /srv/rsync/backup
comment = Backup target
read only = false
list = true
hosts allow = 203.0.113.0/24
hosts deny = *
auth users = deploy
secrets file = /etc/rsyncd.secrets
```

**Command Breakdown & Explanation:**

- `uid` / `gid`: The system user/group the daemon process runs as when
  serving files, once it has dropped root privileges. `nobody`/`nogroup`
  is a safe default for read-only or low-trust modules.
- `use chroot = yes`: Confines the daemon to the module's `path`,
  preventing access to files outside it. Requires the process to still
  have the privileges needed to perform the `chroot()` call.
- `max connections`: Caps simultaneous client connections to this
  daemon instance.
- `[backup]`: Declares a module named `backup`, referenced by clients as
  `host::backup` or `rsync://host/backup`.
- `path = /srv/rsync/backup`: The directory this module serves.
- `read only = false`: Allows clients to upload (write) into this
  module; omit or set to `true` for a strictly read-only mirror.
- `hosts allow` / `hosts deny`: Restricts which client IP addresses or
  subnets may connect to this module, evaluated before any
  username/password check.
- `auth users`: Requires clients to authenticate as one of the listed
  usernames (these are daemon-local usernames, unrelated to system
  accounts) before accessing the module.
- `secrets file`: Path to the file holding daemon usernames and
  passwords for `auth users` (see Section 7.3).

> [!CAUTION]
> `hosts allow` / `hosts deny` alone are not strong access control — IP
> addresses can be spoofed on unprotected networks. Combine them with
> `auth users` and a secrets file for anything beyond a fully public,
> read-only mirror.

### 7.2 Creating the Data Directory and Starting the Daemon

```bash
mkdir -p /srv/rsync/backup
chown nobody:nogroup /srv/rsync/backup
systemctl enable --now rsync
```

**Command Breakdown & Explanation:**

- `chown nobody:nogroup ...`: Ensures the daemon's configured `uid`/`gid`
  can read (and, for writable modules, write) the module path.
- `systemctl enable --now rsync`: On Debian 13, the daemon is managed by
  the same `rsync.service` systemd unit shipped with the `rsync`
  package; enabling it starts `rsyncd` reading `/etc/rsyncd.conf` and
  listening on TCP 873.

### 7.3 Daemon Authentication (Secrets File)

```bash
echo "deploy:CHANGE-ME-STRONG-PASSWORD" > /etc/rsyncd.secrets
chmod 600 /etc/rsyncd.secrets
chown root:root /etc/rsyncd.secrets
```

**Command Breakdown & Explanation:**

- `deploy:CHANGE-ME-STRONG-PASSWORD`: One `username:password` pair per
  line, matching the usernames listed in `auth users` for the relevant
  module.
- `chmod 600`: The daemon refuses to start (or refuses connections
  requiring authentication) if `secrets file` is readable by group or
  other; this is a hard requirement, not just a best practice.

### 7.4 Connecting to a Daemon Module

```bash
rsync -avzP /home/user/documents/ deploy@node01.example.com::backup/documents/
```

**Command Breakdown & Explanation:**

- `deploy@node01.example.com::backup/documents/`: The double-colon
  (`::`) syntax addresses daemon mode directly rather than going through
  SSH. `deploy` must match an entry in `auth users` and
  `rsyncd.secrets`; `rsync` will prompt for the password unless it is
  supplied via the `RSYNC_PASSWORD` environment variable (for scripted,
  unattended use).

> [!TIP]
> The equivalent `rsync://` URL form works identically:
> `rsync -avzP /home/user/documents/ rsync://deploy@node01.example.com/backup/documents/`

## 8. Firewall Considerations

- SSH-based transfers (Sections 3–6) only require the existing SSH port
  (default TCP 22) to be open; no additional firewall rule is needed
  specifically for `rsync`.
- Daemon-mode transfers (Section 7) require TCP port 873 to be reachable
  from the client. If using `nftables` or `ufw` on Debian 13, allow this
  port only from the specific source network(s) listed in `hosts allow`,
  rather than opening it broadly.

```bash
ufw allow from 203.0.113.0/24 to any port 873 proto tcp
```

## 9. Verification and Troubleshooting

### 9.1 Verify rsync daemon service status

```bash
systemctl status rsync
```

What it checks and variables to look for:

- **Active**: Must be `active (running)` if daemon mode is in use.
- **Loaded**: Must be `loaded (/lib/systemd/system/rsync.service; enabled)`.

### 9.2 Verify the daemon is listening on the expected port

```bash
ss -tulnp | grep 873
```

What it checks and variables to look for:

- **State**: Must be `LISTEN` bound to `*:873` or the specific address
  configured, confirming the daemon accepted its configuration and
  bound successfully.

### 9.3 Test daemon module listing without transferring files

```bash
rsync node01.example.com::
```

What it checks and variables to look for:

- **Module list output**: Must show the configured module name(s) (e.g.,
  `backup`) alongside their `comment` text. An empty result or
  connection refusal points to either the daemon not running, a
  firewall block on port 873, or `hosts allow`/`hosts deny` rejecting
  the client's address.

### 9.4 Dry-run a transfer to confirm behavior before executing it

```bash
rsync -avzP --dry-run --delete /home/user/documents/ deploy@203.0.113.10:/srv/backup/documents/
```

What it checks and variables to look for:

- **File list**: Lines prefixed with nothing indicate files that would
  be transferred; lines prefixed `deleting` indicate files that would be
  removed from the destination under `--delete`. No files should
  actually change on either host during a `--dry-run`.

### 9.5 Common issues

- `rsync: connection unexpectedly closed`: Usually indicates an SSH
  authentication failure, or that the remote `rsync` binary is missing
  or an incompatible version. Confirm with a plain `ssh
  user@host` login first.
- `@ERROR: auth failed on module <name>`: The `auth users`/
  `rsyncd.secrets` credentials do not match, or the username was omitted
  from the connection string.
- Permission denied errors on the destination: Check the `uid`/`gid` the
  daemon runs as (daemon mode) or the SSH login user's filesystem
  permissions (SSH mode) against the destination directory's ownership.
- Transfers appear to hang with no progress: Check `--bwlimit` settings
  and confirm the network path allows the port in use (22 for SSH mode,
  873 for daemon mode) — see Section 8.

<!-- Created by: Gergő Téringer, 2026 -->