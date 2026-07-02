# ADDS Installation

```powershell
Install-WindowsFeature -Name Ad-Domain-Services,DNS -IncludeManagementTools
```

## Domain promote without any domain controller

```powershell
Install-ADDSForest -DomainName "contoso.com" -NetbiosName "CONTOSO" # You have to enter your SafeModeAdminPassword twice
```

## Domain promote as RODC into a domain

```powershell
Install-ADDSDomainController -DomainName "contoso.com" -ReadOnly
```

## New domain in a Forest

```powershell
Install-ADDSDomain -NewDomainName "cisco" -ParentDomainName "contoso.com" -DomainType ChildDomain
```
