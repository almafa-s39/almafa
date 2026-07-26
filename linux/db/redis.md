# Redis Key Setup and Scheduled Index Update (Debian 13.3)

This document implements the task shown in the assignment: install Redis, create an initial hard-coded key, build a script that keeps that key updated with the current date/time, and schedule that script to run every two minutes on odd minutes only.

## 1. Install Redis Server

Debian 13's own repositories carry Redis and are sufficient for this task, so the default package is used rather than adding the upstream Redis APT repository.

```bash
apt update
apt install redis-server
systemctl enable --now redis-server
```

**Command Breakdown & Explanation:**

- `apt install redis-server`: Installs the Redis daemon along with `redis-cli`, the command-line client used throughout this document.
- `systemctl enable --now redis-server`: Enables the service to start on boot and starts it immediately in the same command.

Verify the service responds before continuing:

```bash
redis-cli PING
```

This should return `PONG`.

## 2. Create the Initial Key (Hard-Coded)

Per the task, the first entry is created manually with literal, hard-coded values in place of `[date]` and `[current time +1]` — this is a one-time setup step, distinct from the script in Section 3, which will keep the value current going forward.

```bash
redis-cli SET skill39:index "Today is the 2026-07-26 and in one hour is 15:30"
```

**Command Breakdown & Explanation:**

- `SET skill39:index "..."`: Creates (or overwrites) the key `skill39:index` with the given string value.
- The date and time values above are hard-coded placeholders for this manual step only; replace them with today's actual date and the current time plus one hour when running this command, or simply run it once and let the script in Section 3 correct it on its next scheduled run.

## 3. Scheduled Update Script

The script must be named `index_update.*` and placed in `/root`, and must update the same `skill39:index` key with the current date and current time + 1 hour. Either a shell or a PHP version satisfies the task; pick one.

### 3.1 Shell Script Option (`/root/index_update.sh`)

```bash
#!/bin/bash
DATE=$(date +"%Y-%m-%d")
TIME_PLUS_1H=$(date -d "+1 hour" +"%H:%M")
redis-cli SET skill39:index "Today is the $DATE and in one hour is $TIME_PLUS_1H"
```

**Command Breakdown & Explanation:**

- `DATE=$(date +"%Y-%m-%d")`: Captures today's date in `YYYY-MM-DD` format.
- `TIME_PLUS_1H=$(date -d "+1 hour" +"%H:%M")`: Uses GNU `date`'s relative-time parsing to compute the current time plus one hour, in `HH:MM` format.
- `redis-cli SET skill39:index "..."`: Reuses the exact same `SET` command from Section 2, now with dynamically computed values instead of hard-coded ones.

```bash
chmod +x /root/index_update.sh
```

This makes the script directly executable, which is required for the cron entry in Section 4.

### 3.2 PHP Script Option (`/root/index_update.php`)

PHP's Redis extension is required if this option is used instead of the shell script:

```bash
apt install php-cli php-redis
```

```php
#!/usr/bin/php
<?php
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$date = date("Y-m-d");
$time_plus_1h = date("H:i", strtotime("+1 hour"));

$redis->set('skill39:index', "Today is the $date and in one hour is $time_plus_1h");
```

**Command Breakdown & Explanation:**

- `$redis->connect('127.0.0.1', 6379)`: Connects to the locally running Redis instance on its default port.
- `date("H:i", strtotime("+1 hour"))`: PHP's equivalent of the shell script's relative-time calculation, producing the same `HH:MM` current-time-plus-one-hour value.
- `$redis->set('skill39:index', ...)`: Equivalent of the `redis-cli SET` command, callable directly from PHP without shelling out. The task also allows reading the value back with `$redis->get('skill39:index');`, if needed for testing.

```bash
chmod +x /root/index_update.php
```

## 4. Schedule the Script (Every Two Minutes, Odd Minutes Only)

"Every two minutes on odd minutes" means the script must run at minute 1, 3, 5, 7 ... 59 of every hour — never on an even minute. Cron's step syntax combined with a range starting at an odd number produces exactly this pattern.

```bash
crontab -e
```

Add one of the following lines, depending on which script was built in Section 3:

```cron
1-59/2 * * * * /root/index_update.sh >> /var/log/index_update.log 2>&1
```

```cron
1-59/2 * * * * /usr/bin/php /root/index_update.php >> /var/log/index_update.log 2>&1
```

**Command Breakdown & Explanation:**

- `1-59/2`: Minute field expressing "starting at minute 1, every 2 minutes, up to minute 59" — this lands only on odd minutes (1, 3, 5, ... 59), satisfying the task's requirement exactly.
- `* * * *` (remaining fields): Every hour, every day of month, every month, every day of week — no further restriction beyond the minute field.
- `>> /var/log/index_update.log 2>&1`: Redirects both standard output and errors to a log file, since cron jobs otherwise fail silently from the user's perspective.

## 5. Verification and Troubleshooting

### 5.1 Verify Redis is running

```bash
systemctl status redis-server
```

What it checks and variables to look for:

- **Active**: Must be `active (running)`.

### 5.2 Verify the key exists and updates correctly

```bash
redis-cli GET skill39:index
```

What it checks and variables to look for:

- **Value**: Must contain a real date and a time exactly one hour ahead of the current time. Run this command twice, roughly two minutes apart, to confirm the value actually changes between scheduled runs rather than being stale.

### 5.3 Verify the cron job is registered

```bash
crontab -l
```

What it checks and variables to look for:

- **Output**: Must show the exact `1-59/2 * * * * ...` line added in Section 4. If missing, the job was not saved correctly by `crontab -e`.

### 5.4 Verify the script is actually executing on schedule

```bash
tail -f /var/log/index_update.log
grep CRON /var/log/syslog
```

What it checks and variables to look for:

- **`index_update.log`**: Should stay empty on success (no errors were redirected into it) if the script produces no output; any PHP or `redis-cli` error text here points directly at the failure.
- **`/var/log/syslog` CRON entries**: Should show a new line every two minutes at odd-minute timestamps (`:01`, `:03`, `:05`, ...), confirming cron itself is invoking the job on the intended schedule.