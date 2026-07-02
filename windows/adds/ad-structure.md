# AD Structure settings

## Ou

```powershell
New-ADOrganizationalUnit -Name IT -Path DC=contoso,DC=com
```

## Group

```powershell
New-ADGroup
```

## User

```powershell
# Variables
$username = sysadmin
$upnSuffix = contoso.com
$securePass = ConvertTo-SecureString "YourPassw0rd!" -AsPlainText -Force
$ouPath = ou=IT,dc=contoso,dc=com

# Create new user
New-ADUser -Name $username `
    -SamAccountName $username `
    -UserPrincipalName "$username@$upnSuffix" `
    -Path $ouPath `
    -AccountPassword $securePass `
    -Enabled $true `
    -PasswordNeverExpires $true

Add-ADGroupMember -Identity IT -Members $username
```

## FGPP

```powershell
# Create the Fine Grained Password Policy
New-ADFineGrainedPasswordPolicy -Name "IT-PSO" `
    -Precedence 10 `
    -ComplexityEnabled $true `
    -MinPasswordLength 16

# 2. Apply the PSO to Security Group
Add-ADFineGrainedPasswordPolicySubject -Identity "IT-PSO" -Subjects "IT"
```