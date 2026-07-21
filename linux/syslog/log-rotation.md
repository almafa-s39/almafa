<!-- 
---
title: "Log rotation and archive"
author: "Gergő Téringer"
---
 -->
# Log rotation and archive

## Log rotation

### Time based log rotation

This one is tricky, because all other log rotation is included in syslog-ng. If you want to configure time based you have to use the Debians built-in log rotation. You have to set up a timer service, or a cronjob, to achieve the timing of the log rotation.

Options:

- **Path**: Assign a path to use this rotation (you can use the wildcard character)
- **rotate**: How much rotated files to keep.
- **trigger**: hourly / daily / weekly / monthly / yearly / size [SIZE] / minsize [SIZE] / maxsize [SIZE]
- **missingok**: If the file is missing it goes to the next one without issue
- **nomissingok**: If the file is missing it throws an error
- **ifempty**: Rotate the log file even if it is empty
- **noifempty**: Doesn't rotate the log file if it is empty
- **create [mode] [owner] [group]**: Immediately creates a new, empty log file after rotation with the specified permissions
- **copytruncate**: Truncates the original log file in place after creating a copy, instead of moving the old log file. Useful when applications cannot be told to close their log file.
- **compress**: Old versions of log files are compressed (by default, using gzip).
- **nocompress**: Old versions of log files are not compressed.
- **delaycompress**: Postpones compression of the previous log file to the next rotation cycle. This is often used with compress and is helpful when programs might still be writing to the old log file immediately after rotation.
- **dateext**: Archives old versions of log files adding a date extension like YYYYMMDD instead of simply adding a number (like .1, .2).
- **prerotate**: You have to close it with **endscript**. It runs the script that is inside the tags before rotation.
- **postrotate**: You have to close it with **endscript**. It runs the script that is inside the tags after rotation. Reload the syslog service you're using (or other app if you just using it for OpenVPN etc..)!
- **sharedscripts**: The pre- and/or postrotate block runs only one time (without it runs to every conf file)

> [!NOTE]
> Put these files under `/etc/logrotate.d/` directory, by the name you want.

```bash
/var/log/myapp/*.log {
    daily
    rotate 7
    size 50M
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp myapp
    sharedscripts
    postrotate
        /bin/systemctl reload myapp.service > /dev/null 2>/dev/null || true
    endscript
}
```

#### Crontab settings

You will implement the timer by cron, because it is enough for us that we can rotate our logs by minutes. Enter cron settings with `crontab -e` (with root user), and use the following line to rotate your logs in every hour and in every 6 hours.

```bash
0 *   * * * /usr/sbin/logrotate /etc/logrotate.d/dns
0 */6 * * * /usr/sbin/logrotate /etc/logrotate.d/vpn
```

## Archive

### Script

`touch /usr/local/bin/archive.sh; chmod +x /usr/local/bin/archive.sh`

```bash
#!/bin/bash
PATHS="/log/dns/*/*.log.*  /log/vpn/*/*.log.*"
mkdir -p /archive
tar -czf /archive/logs_$(date -d "yesterday" +\%Y\%m\%d).tar.gz "$PATHS" 2>/dev/null
rm -rf "$PATHS"
```

### Cronjob

Add a new cron job, which will run this time, on daily basis.

```bash
1 0   * * * /usr/local/bin/archive.sh
```

<!-- Created by: Gergő Téringer, 2026 -->