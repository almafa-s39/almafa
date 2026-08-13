```powershell
New-NetFirewallRule -DisplayName "Block Outbound Port 80" -Direction Outbound -LocalPort 80 -Protocol TCP -Action Block -Profile Public
```

The `-Profile` option:

- Specifies one or more profiles to which the rule is assigned. The rule is active on the local computer only when the specified profile is currently active. This relationship is many-to-many and can be indirectly modified by the user, by changing the Profiles field on instances of firewall rules.
- **Separate multiple entries with a comma and do not include any spaces.** e.g. `-Profile Domain,Private`
- Default value: `Any`
- Possible values: `Any`, `Domain`, `Private`, `Public`, `NotApplicable`
