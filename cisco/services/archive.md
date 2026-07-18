# Cisco Archive

The Cisco IOS archive feature provides a built-in version control system for device configurations. It allows administrators to automatically save backups of the running configuration locally or remotely, ensuring that previous working states can be easily restored, downloaded, or compared against current setups.

## 1. Automated Local Backups

The provided configuration establishes an automated local backup system on the router's flash memory. It is designed to capture configuration states based on both administrative actions and a continuous schedule, while automatically managing storage space.

**Configuration:**

```cisco
archive
 path flash:wsc-backup-$h-
 maximum 5
 write-memory
 time-period 60
```

**Command Breakdown & Explanation:**

- `archive`: Enters the archive configuration mode where version control parameters are defined.
- `path flash:wsc-backup-$h-`: Defines the destination storage location and the file naming convention. The `flash:` keyword specifies the local memory. The `$h` is a system variable that dynamically inserts the configured hostname of the device. The trailing hyphen ensures a clean visual separation before the system automatically appends a version number (for example, resulting in `wsc-backup-R1-1`, `wsc-backup-R1-2`).
- `maximum 5`: Limits the archive directory to strictly the 5 most recent configuration files. When a 6th backup is generated, the oldest file is automatically deleted, which prevents the flash memory from filling up over time.
- `write-memory`: Instructs the router to instantly generate and save a new archive file every single time an administrator issues a `write memory` or `copy running-config startup-config` command.
- `time-period 60`: Establishes a recurring timer to automatically save a backup every 60 minutes, acting as a continuous failsafe regardless of manual saves.

Practical Example:
If an administrator makes an extensive configuration change but forgets to save it before a sudden power failure, the `time-period 60` command ensures you have a backup that is at most one hour old. Conversely, if a breaking change is intentionally saved, the `write-memory` trigger guarantees that a snapshot of the exact moment before the change is safely stored in the `flash:` directory, allowing you to use the `configure replace` command to instantly roll back to a known-good state.

## 2. Troubleshooting

Verifying the Cisco Archive feature involves checking the current status of saved configurations, comparing files to see what changes were made, and executing rollbacks if a configuration breaks the network.

### 2.1 Verifying Saved Archives

This command lists all configurations currently stored by the archive process, providing a quick inventory of available rollback points.

**Command:** `show archive`

What it checks and variables to look for:

- `Maximum Archive Configurations`: Confirms the limit you set (e.g., `5`).
- `Archive #` / `Name`: Lists the sequential index number and the exact file path/name of each saved configuration on the flash drive (e.g., `flash:wsc-backup-R1-1`).
- `Next Archive File`: Shows the filename that will be generated the next time a save is triggered.
- `Most Recent Archive File`: Indicates your latest successful backup, which is typically your safest rollback point.

### 2.2 Comparing Configurations

Before performing a rollback, it is critical to see exactly what changed between two configuration files (or between an archive and the current running config).

**Command:** `show archive config differences flash:wsc-backup-R1-1 system:running-config`

What it checks and variables to look for:

- `+` (Plus sign): Indicates commands that are present in the second file (the running config in this example) but missing from the first file. These are additions made since the backup.
- `-` (Minus sign): Indicates commands that were in the original backup but have been removed from the current running config.
- `Contextal lines`: The command outputs the interface or routing process block to give context to where the changes occurred.

### 2.3 Restoring an Archive Configuration

If a change has broken connectivity or caused routing issues, you can instruct the router to automatically replace the current running configuration with a saved archive without needing to reboot.

**Command:** `configure replace flash:wsc-backup-R1-1`

What it checks and variables to look for:

- `Total number of passes`: The router analyzes the difference between the files and applies the changes in passes (adding new commands, negating removed commands).
- `Rollback Successful`: Confirms the router has completely synchronized the running config with the chosen archive file.
- `Time elapsed`: How long the replacement process took. (Note: During this time, minor packet loss may occur depending on the severity of the replaced configurations, such as routing protocol restarts).
