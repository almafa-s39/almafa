# Logging

System logging is a fundamental component of network monitoring, troubleshooting, and security auditing. A properly configured logging mechanism ensures that critical events are captured locally and forwarded reliably to a centralized log management server.

## 1. Log Timestamps

Accurate timestamps are crucial for correlating events across multiple devices during an incident investigation. By default, Cisco routers might use uptime or lack timezone context, making historical troubleshooting extremely difficult.

**Configuration:**

```cisco
service timestamps log datetime msec localtime show-timezone
service timestamps debug datetime msec localtime show-timezone
```

**Command Breakdown & Explanation:**

- `service timestamps log`: Applies the timestamp format to standard system logs.
- `service timestamps debug`: Applies the same timestamp format to debugging output.
- `datetime msec`: Formats the timestamp to include the date and time down to the millisecond. This is vital during rapid sequences of events, such as routing protocol convergence or spanning-tree topology changes, where multiple actions occur within a single second.
- `localtime`: Instructs the router to use the locally configured time (such as the CET/CEST timezone configured earlier) instead of the default UTC.
- `show-timezone`: Appends the timezone acronym to the log entry so the reader or the log parser knows exactly what time reference is being used.

---

## 2. Local Buffer and External Syslog Server

Logs can be stored in the device's local memory (RAM) and sent to an external Syslog server over the network. Best practices dictate keeping highly detailed logs locally for deep CLI troubleshooting while sending operational events to the central server to avoid overwhelming the network with debug traffic.

**Configuration:**

```cisco
logging buffered 16384 debugging
logging host 10.20.200.100
logging trap informational
logging source-interface Vlan103
do wr
```

**Command Breakdown & Explanation:**

- `logging buffered 16384 debugging`: Allocates `16384` bytes (16 KB) of RAM to store logs locally. The `debugging` keyword sets the severity level to 7, meaning everything from critical hardware failures to highly verbose debug messages will be stored in this local ring buffer. Once the buffer fills, the oldest entries are overwritten.
- `logging host 10.20.200.100`: Defines the IP address of the external Syslog server (e.g., an ELK stack, Splunk, or a dedicated NMS).
- `logging trap informational`: Sets a severity filter for the logs sent to the external server. The `informational` keyword (severity level 6) means the router will forward all logs from level 0 (Emergencies) up to level 6, but will explicitly exclude level 7 (Debugging) logs. This prevents debug floods from impacting CPU, bandwidth, or external storage.
- `logging source-interface Vlan103`: Forces the router to use the IP address of `Vlan103` as the source IP for all outgoing Syslog packets. This is critical for centralized logging environments, as it ensures the Syslog server always recognizes and categorizes the router by a consistent IP address, regardless of which physical interface the packet exits.
- `do wr`: A shortcut for `do write memory`. Because you are in global configuration mode, the `do` keyword allows you to execute privileged exec commands to save the running configuration to the startup configuration (NVRAM).

**Practical Example:**
If an OSPF neighbor goes down, the router generates a syslog message. Because of this configuration, the log will feature a precise local timestamp, it will be immediately saved in the router's local 16KB memory buffer, and a copy will be forwarded to `10.20.200.100` originating from the IP of `Vlan103`.
