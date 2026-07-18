# Cisco EEM

Cisco Embedded Event Manager (EEM) is a powerful on-device automation tool. It allows network engineers to write scripts (applets) that monitor the router for specific events, such as syslog messages, interface counters, or timers, and automatically trigger a sequence of CLI commands or actions in response.

## Examples

### 1. Syslog-Triggered Applet

The provided configuration demonstrates a fundamental EEM use case: watching for a specific syslog message and generating a custom, high-priority alert when that message occurs.

**Configuration:**

```cisco
event manager applet LINK-WATCH
 event syslog pattern "Interface GigabitEthernet0/3, changed state to down"
 action 1.0 syslog priority critical msg "EEM: ISP uplink is DOWN"
```

**Command Breakdown & Explanation:**

- `event manager applet LINK-WATCH`: Creates an EEM applet named `LINK-WATCH` and enters applet configuration mode.
- `event syslog pattern "Interface GigabitEthernet0/3, changed state to down"`: Defines the trigger. The applet constantly scans internal syslog generation. If it sees a log message that exactly matches the string inside the quotes, the applet executes.
- `action 1.0 syslog priority critical msg "EEM: ISP uplink is DOWN"`: Defines the action to take when triggered. The `1.0` is a sequence number, which allows you to chain multiple actions together in order (e.g., `action 2.0`, `action 3.0`). This specific action generates a new syslog message with a `critical` severity level, containing your custom string.

**Practical Example:**
If `GigabitEthernet0/3` is your primary ISP connection, a standard interface down message might get lost in a sea of normal informational logs. By using EEM to catch this specific event and generate a `critical` log, your external monitoring system can be configured to page the on-call engineer immediately upon seeing the `EEM: ISP uplink is DOWN` string, ensuring faster incident response.

### 2. Automated Configuration Backup on Save

A common operational risk is saving a configuration locally but forgetting to back it up to a centralized server. This applet intercepts the `write memory` command and automatically triggers a TFTP backup right after saving.

**Configuration:**

```cisco
event manager applet AUTO-BACKUP
 event cli pattern "write memory" sync yes
 action 1.0 cli command "enable"
 action 2.0 cli command "write memory"
 action 3.0 cli command "copy running-config tftp://10.20.200.100/router-config.txt" pattern "Address"
 action 3.1 cli command ""
 action 4.0 syslog msg "EEM: Configuration automatically backed up to TFTP"
```

**Command Breakdown & Explanation:**

- `event cli pattern`: Instructs EEM to look for the exact CLI command typed by the user.
- `sync yes`: Tells the router to hold the original command and let the EEM applet decide whether to run it or not.
- `action 1.0 cli command "enable"`: EEM runs in a virtual VTY line, so it must enter privileged exec mode first before it can run administrative commands.
- `action 2.0 cli command "write memory"`: Since we intercepted the user's save command with `sync yes`, we must explicitly execute the save action inside the script so the local NVRAM is updated.
- `action 3.0` and `action 3.1`: These execute the copy command to your management server. The `pattern "Address"` looks for the router's CLI prompt asking to confirm the destination IP, and the empty string in `3.1` simulates pressing the Enter key to accept it.

### 3. Periodic CPU Utilization Logging

Intermittent CPU spikes are notoriously difficult to troubleshoot because they often disappear before an engineer can log in. This applet uses a timer to run a diagnostic command every 5 minutes and appends the output to a local file.

**Configuration:**

```cisco
event manager applet CPU-MONITOR
 event timer watchdog time 300
 action 1.0 cli command "enable"
 action 2.0 cli command "show processes cpu sorted | append flash:cpu-history.txt"
```

**Command Breakdown & Explanation:**

- `event timer watchdog time 300`: Sets the trigger to run on a repeating timer every 300 seconds (5 minutes).
- `action 2.0`: Runs the CPU checking command and uses the `| append` pipe to add the text output to a file named `cpu-history.txt` directly on the router's flash memory.

**Practical Example:**
If users report that the network was unexpectedly slow at 2:00 AM, you can log in the next morning, review the `cpu-history.txt` file, and see exactly which process was consuming resources at that specific time without needing a live session.

## Troubleshooting

### 1. Verifying Registered Applets

This command displays all currently active EEM applets and policies, confirming that the router has successfully parsed and loaded your automation script into memory.

**Command:** `show event manager policy registered`

What it checks and variables to look for:

- `No.` / `Class` / `Type`: Indicates the policy classification (e.g., `applet`).
- `Event Type`: Shows the trigger mechanism tied to the applet, such as `syslog`, `cli`, or `timer`.
- `Name`: The configured name of your applet (e.g., `LINK-WATCH`). If your applet is missing from this list, there is a syntax error in your configuration preventing it from registering.

### 2. Reviewing Applet Execution History

If you suspect an event occurred but the expected action didn't happen, the history log will tell you exactly when EEM triggered and if it encountered any immediate issues.

**Command:** `show event manager history events`

What it checks and variables to look for:

- `Job ID`: A unique tracking number for the specific execution instance.
- `Name`: The name of the applet that was triggered.
- `Trigger`: The exact time the event was detected.
- `Success`: Indicates if the applet ran successfully to completion. If you see `Fail`, it usually means a CLI action within the script hung up (e.g., waiting for an unhandled prompt) or encountered a syntax error.

### 3. Real-Time EEM Debugging

When an applet is failing silently, enabling real-time debugging allows you to watch the EEM process run exactly as if it were a human typing the commands in the background.

**Command:** `debug event manager action cli`

What it checks and variables to look for:

- `cli_exec`: Shows the exact command being passed to the router's virtual terminal line.
- `cli_read`: Shows the router's response to the command. This is vital for troubleshooting interactive prompts. For example, if your copy command asks `Destination filename [router-config.txt]?` and your script does not send a carriage return (the empty string `""`), the debug will show the script hanging at this prompt and timing out.
