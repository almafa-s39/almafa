# ADDS Installation

```powershell
Install-WindowsFeature -Name Ad-Domain-Services,DNS -IncludeManagementTools
```

## Domain promote without any domain controller

```powershell
Install-ADDSForest -DomainName "france.net" -NetbiosName "FRANCE" # You have to enter your SafeModeAdminPassword twice
```

## Domain promote as RODC into a domain

```powershell
Install-ADDSDomainController -DomainName "france.net" -ReadOnly
```

## New domain in a Forest

```powershell
Install-ADDSDomain -NewDomainName "paris" -ParentDomainName "france.net" -DomainType ChildDomain
```
