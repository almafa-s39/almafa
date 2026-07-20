# Windows Remote Event Viewer

Enable the following Net Firewall Rule, to check the event log of another computer.

```powershell
# Enable that Rule
Enable-NetFirewallRule -DisplayGroup "Remote Event Log Management"

# Disable the whole firewall
Set-NetFirewallProfile Domain,Private,Public -Enabled False
```
