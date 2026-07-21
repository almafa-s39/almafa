<!-- 
---
title: "WEF"
author: "Gergő Téringer"
---
 -->
# WEF

## 1. Windows Event Forwarding (WEF) over HTTPS configuration

Windows Event Forwarding (WEF) allows you to centralize event logs from multiple endpoints to a single collector server. This guide configures a "Source-Initiated" subscription over HTTPS, which is highly recommended for modern Active Directory environments, especially when endpoints are remote or traversing firewalls. In this architecture, the source computers check in with the collector (SRV2) to retrieve their subscription configurations and push their logs securely.

## 1.1 Group Policy Configuration (Source Computers)

To allow endpoints to push their logs, they must be configured to read their own security logs, run the WinRM service, and know the URL of the centralized subscription manager.

Create a GPO applied to your source endpoints with the following configurations:

1. Grant Network Service Access to Event Logs:
   Computer Configuration > Policies > Windows Settings > Security Settings > Restricted Groups
   - Add the group **Event Log Readers** and set its members to include **NETWORK SERVICE**.

2. Enable WinRM Service:
   Computer Configuration > Policies > Windows Settings > Security Settings > System Services
   - Find **Windows Remote Management (WS-Management)** and set it to **Automatic** startup.

3. Configure the Subscription Manager URL:
   Computer Configuration > Policies > Administrative Templates > Windows Components > Event Forwarding > Configure target Subscription Manager
   - Set to Enabled.
   - Add the URI: `Server=https://SRV2.skillsnet.dk:5986/wsman/SubscriptionManager/WEC,Refresh=60`

4. Grant Read Access to the Security Log (SDDL string):
   Computer Configuration > Policies > Administrative Templates > Windows Components > Event Log Service > Security > Configure Log Access
   - Set to Enabled.
   - Enter the following SDDL string:
     `O:BAG:SYD:(A;;0xf0005;;;SY)(A;;0x5;;;BA)(A;;0x1;;;S-1-5-20)(A;;0x1;;;S-1-5-32-573)`

> [!NOTE]
> The SDDL string explicitly grants the NETWORK SERVICE account (S-1-5-20) and the Event Log Readers group (S-1-5-32-573) read permissions (0x1) to the Security Event Log, which is otherwise heavily restricted by Windows.
> [!WARNING]
> Do not excessively modify these WEF GPOs during testing. Windows aggressively caches Event Forwarding configurations in the registry. If you make a mistake, endpoints may stubbornly hold onto the broken configuration, requiring you to manually clear the registry on every target machine. Get the policy right the first time.

## 1.2 Creating the Subscription XML Template

Instead of configuring the subscription manually on a Server Core machine, it is easier to create the subscription using the Event Viewer GUI on a machine with a Desktop Experience, export it to XML, modify the XML, and import it onto the target Server Core collector.

```powershell
wecutil gs "Subscription Name" /f:xml > C:\subscription.xml
```

**Command Breakdown & Explanation:**

- `wecutil gs`: Gets the configuration settings of a specific WEF subscription.
- `/f:xml`: Formats the output as an XML file so it can be easily modified and re-imported.

> [!IMPORTANT]
> Open the generated XML file in a text editor and add the line `<ConfigurationMode>Custom</ConfigurationMode>` right after the opening tags. This forces the collector to accept the custom refresh intervals configured in your GPO. Once edited, transfer this file to your collector server (SRV2). Ensure the wecsvc service is temporarily stopped on the collector before proceeding.

## 1.3 Configuring the Event Collector (SRV2)

The collector server needs to request an auto-enrollment certificate to secure the HTTPS WinRM listener, initialize the Event Collector service, and import the XML configuration you transferred. Run these commands on SRV2.

```powershell
# Force group policy update to pull the computer certificate for HTTPS
gpupdate /force

# Configure WinRM to listen on HTTPS (Port 5986)
winrm qc -transport:https

# Quick-configure the Windows Event Collector service
wecutil qc

# Import the subscription XML file
wecutil cs .\log.xml

# Start and enable the Windows Event Collector service
Start-Service wecsvc
Set-Service wecsvc -StartupType Automatic

# Open the firewall for inbound Event Log Management
Enable-NetFirewallRule -DisplayGroup "Remote Event Log Management"
```

**Command Breakdown & Explanation:**

- `winrm qc -transport:https`: Quick-configures the WinRM service and automatically searches the local machine's personal certificate store for a valid SSL certificate with Server Authentication OID to bind to port 5986.
- `wecutil qc`: Quick-configures the WEF service, ensuring the `ForwardedEvents` log channel is enabled and properly dimensioned.
- `wecutil cs`: Creates a Subscription (`cs`) from the provided XML file.

## 2. Troubleshooting and Verification

Verifying WEF involves checking the runtime status of the subscription to see if endpoints are successfully communicating with the collector server.

### 2.1 Verifying WEF Subscription Runtime Status

Run this command on the collector server (SRV2) to see if source computers are successfully checking in.

**Command:** `wecutil gr "Subscription Name"`

**What it checks and variables to look for:**

- **RunTimeStatus**: Should read `Active`. If it reads `Error`, the collector cannot process the incoming logs.
- **LastError**: Look for error codes like `0x80338126` (which indicates the endpoint does not have permission to read the requested log) or `0x80338012` (which indicates a WinRM HTTPS certificate validation failure).
- **EventSources**: Lists the FQDNs of the endpoints that have successfully checked in to the collector. If this is empty, check the `WindowsRM` or `Eventlog-ForwardingPlugin` event logs on the source computer to find out why it cannot reach the collector.

<!-- Created by: Gergő Téringer, 2026 -->