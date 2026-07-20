# Active Directory Domain Services (AD DS) Installation & Promotion

## Prerequisites

Before promoting a server to a Domain Controller, ensure that the server has a **static IP address** configured. If you are adding a Read-Only Domain Controller (RODC) or a Child Domain, the server's primary DNS must point to an existing Domain Controller in the forest.

## 1. Install AD DS and DNS Roles

First, install the Active Directory Domain Services and DNS Server roles, along with their associated management tools.

```powershell
# Install the AD DS and DNS features
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
```

## 2. Domain Promotion Scenarios

Choose the appropriate scenario below based on your deployment topology.

### Scenario A: Create a New Forest (First Domain Controller)

Use this command when setting up a brand new environment from scratch.

```powershell
# Promote the server to a Domain Controller in a new forest
# Note: You will be prompted to enter the SafeModeAdministratorPassword (DSRM password) twice.
Install-ADDSForest -DomainName "contoso.com" -NetbiosName "CONTOSO" -InstallDns -Force
```

### Scenario B: Add a Read-Only Domain Controller (RODC) to an Existing Domain

Use this command to deploy an RODC in a branch office or perimeter network.

```powershell
# Promote the server as an RODC
# Note: You will be prompted for Domain Admin credentials and the DSRM password.
Install-ADDSDomainController -DomainName "contoso.com" -ReadOnly -Force
```

### Scenario C: Create a New Child Domain in an Existing Forest

Use this command to create a new sub-domain (e.g., cisco.contoso.com) under an existing parent domain.

```powershell
# Promote the server to create a new child domain
# Note: You will be prompted for Enterprise Admin credentials and the DSRM password.
Install-ADDSDomain -NewDomainName "cisco" -ParentDomainName "contoso.com" -DomainType ChildDomain -Force
```

## 3. Verification

Once the server reboots after promotion, log in with domain administrator credentials and use the following commands to verify the health and status of your Active Directory environment:

```powershell
# Verify the local Domain Controller configuration and status
Get-ADDomainController -Server $env:COMPUTERNAME

# View the details of the Domain (e.g., Domain Mode, PDC Emulator)
Get-ADDomain

# View the details of the entire Forest (e.g., Schema Master, Forest Mode)
Get-ADForest

# Check the SYSVOL and NETLOGON shares to ensure they are published
net share
```

## 4. Troubleshooting

If the promotion fails or the Domain Controller is not functioning correctly, use these steps to diagnose the issue:

```powershell
# 1. Run the Domain Controller Diagnostic Tool to check for errors in AD components
dcdiag /q

# 2. Verify that the DNS Server service is running and resolving the domain
Get-Service DNS
Resolve-DnsName -Name "contoso.com"

# 3. Check for replication issues (applicable for RODC or multiple DCs)
repadmin /showrepl

# 4. Pre-test a promotion without actually applying changes (Useful before running Install-ADDSDomainController)
Test-ADDSDomainControllerInstallation -DomainName "contoso.com" -ReadOnly

# 5. Check the Directory Services event log for critical errors during or after promotion
Get-WinEvent -LogName "Directory Service" -MaxEvents 20 | Where-Object {$_.LevelDisplayName -eq "Error"}
```
