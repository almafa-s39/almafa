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
