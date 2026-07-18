# AAA

Authentication, Authorization, and Accounting (AAA) provides a highly scalable framework for securing device access. In an enterprise environment, relying on local usernames is difficult to manage. Integrating AAA with remote RADIUS or TACACS+ servers ensures centralized credential management and auditing.

## 1. Radius

RADIUS is an open-standard protocol commonly used for network access. It encrypts only the password in the access-request packet, leaving the rest of the payload unencrypted. It combines authentication and authorization into a single process.

**Configuration:**

```cisco
! 1. Enable AAA
aaa new-model

! 2. Define the RADIUS server
radius server SVR1
 address ipv4 10.20.200.100 auth-port 1812 acct-port 1813
 key Passw0rd

! 3. Authentication: RADIUS primary, Local fallback
aaa authentication login default group radius local

! 4. Authorization: RADIUS primary, Local fallback (for EXEC shell access)
aaa authorization exec default group radius local

! 5. Accounting: Send Login and EXEC session records to RADIUS
aaa accounting exec default start-stop group radius
aaa accounting connection default start-stop group radius

! 6. Ensure VTY lines use the default AAA lists
line vty 0 15
 login authentication default
 authorization exec default
 accounting exec default
 accounting connection default
end
```

**Command Breakdown & Explanation:**

- `aaa new-model`: Globally enables the AAA architecture on the Cisco device.
- `radius server SVR1`: Creates a RADIUS server profile named `SVR1`.
- `address ipv4 ...`: Specifies the server IP address and explicitly defines the standard RADIUS UDP ports (`1812` for authentication, `1813` for accounting).
- `key Passw0rd`: Defines the shared secret key used to encrypt the password exchange. This must match the server configuration.
- `aaa authentication login default ...`: Dictates that the router will first attempt to authenticate users against the RADIUS server. If the server is unreachable, it will fall back to the `local` user database.
- `aaa authorization exec default ...`: Determines if a successfully authenticated user is allowed to start an exec shell session based on the RADIUS response.
- `aaa accounting ...`: Instructs the router to send `start` and `stop` records to the RADIUS server, logging exactly when a user connects and disconnects.
- `line vty 0 15`: Enters the configuration for all remote virtual terminal lines (SSH/Telnet) and explicitly applies the `default` AAA lists configured above.

**Practical Example:**
When a junior engineer attempts to SSH into the router, the router sends their credentials to `10.20.200.100`. The RADIUS server verifies the credentials and returns an accept message. The router then generates an accounting start record so the organization has a precise timestamp of the login event.

## 2. TACACS+

TACACS+ is a Cisco-proprietary protocol (though widely supported) that separates authentication, authorization, and accounting into distinct processes. Unlike RADIUS, TACACS+ encrypts the entire payload of the packet, making it the preferred choice for device administration security. It also allows for highly granular command-by-command authorization.

**Configuration:**

```cisco
! 1. Enable AAA
aaa new-model

! 2. Define the TACACS+ server
tacacs server SVR2
 address ipv4 10.20.200.101
 key Passw0rd

! 3. Authentication: TACACS+ primary, Local fallback
aaa authentication login default group tacacs+ local

! 4. Authorization: TACACS+ primary, Local fallback
aaa authorization exec default group tacacs+ local

! 5. Command Authorization: Verify specific commands
aaa authorization commands 15 default group tacacs+ none

! 6. Accounting: Send Login and EXEC session records
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
```

**Command Breakdown & Explanation:**

- `tacacs server SVR2`: Creates a TACACS+ server profile named `SVR2`.
- `address ipv4 ...`: Specifies the IP address of the authentication server. TACACS+ uses TCP port 49 by default.
- `aaa authentication login default ...`: Sets the primary login method to TACACS+, falling back to `local` only if the server times out.
- `aaa authorization commands 15 default ...`: This is a massive advantage of TACACS+. It forces the router to contact the server every single time a user types a privilege level 15 command. If the server says the user is not allowed to run that specific command, the router rejects it.
- `aaa accounting commands 15 default ...`: Instructs the router to send an audit log to the TACACS+ server for every single privilege level 15 command executed, creating a complete administrative trail.

**Practical Example:**
If you want to allow a specific support team to use `show run` but block them from using `configure terminal`, TACACS+ handles this seamlessly. When the user types `configure terminal`, the router pauses, asks the TACACS+ server for permission, receives a denial, and blocks the command. RADIUS cannot do this natively.

## 3. Troubleshooting

Verifying AAA involves confirming that the router can reach the configured servers, that the shared secret keys match, and observing the authentication process to pinpoint exactly where a failure occurs.

### 3.1 Verifying AAA Server Reachability and Status

This command verifies the operational state of the configured RADIUS and TACACS+ servers, including how many requests have been sent and if the servers are responding.

**Command:** `show aaa servers`

What it checks and variables to look for:

- `Server`: The IP address and port of the configured server.
- `State`: Look for `UP`. If it says `DEAD`, the router has marked the server as unreachable.
- `Requests` / `Timeouts`: If the `Requests` counter increments but the `Timeouts` counter also rises rapidly, there is a network reachability issue (e.g., a firewall blocking UDP 1812 or TCP 49) or the server is offline.
- `Bad authenticators`: A high number here almost always indicates a mismatched shared secret key between the router and the AAA server.

### 3.2 Testing Authentication Manually

Instead of opening a completely new SSH session and risking getting locked out, you can simulate an authentication request directly from the privileged exec prompt to verify your configuration.

**Command:**

```cisco
test aaa group radius admin Passw0rd legacy
test aaa group tacacs+ admin Passw0rd legacy
```

What it checks and variables to look for:

- `Attempted login to ...`: Confirms which server IP the router is targeting.
- `User successfully authenticated`: Indicates a successful connection and key match.
- `User authentication failed`: Indicates reachability is fine, but the server actively rejected the credentials (wrong username, password, or account disabled).
- `Timeout`: Indicates a routing issue, firewall block, or downed server.

### 3.3 Real-Time AAA Debugging

If the `test aaa` command fails and you need to see the exact packet exchange, debugging will reveal the specific stage where the process breaks down.

**Command:**

```cisco
debug aaa authentication
debug tacacs
debug radius
```

What it checks and variables to look for:

- `GETUSER`: Shows the router successfully prompting for the username.
- `GETPASSWORD`: Shows the router prompting for the password.
- `PASS_ADD`: Indicates the credentials are being packaged to send.
- For RADIUS: Watch for `Access-Accept` (success) or `Access-Reject` (failure/wrong credentials).
- For TACACS+: Watch for `PASS` (success) or `FAIL` (wrong credentials). If you do not see these responses coming back from the server, verify network connectivity and the shared secret key.
