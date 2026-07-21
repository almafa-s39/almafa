<!-- 
---
title: "Active Directory Structure Configuration"
author: "Gergő Téringer"
---
 -->
# Active Directory Structure Configuration

## Prerequisites

This guide assumes you are running these commands on a Domain Controller or a machine with the **Active Directory module for Windows PowerShell** installed (RSAT). Ensure you are running PowerShell as an Administrator with Domain Admin privileges.

## 1. Organizational Units (OUs)

Create a logical structure by defining an Organizational Unit. Be sure to replace `DC=contoso,DC=com` with your actual domain distinguished name.

```powershell
# Create the IT Organizational Unit at the domain root
New-ADOrganizationalUnit -Name "IT" -Path "DC=contoso,DC=com"
```

## 2. Security Groups

Create a security group within the newly created OU to manage permissions and policy assignments.

```powershell
# Create a Global Security Group named 'IT' inside the IT OU
New-ADGroup -Name "IT" -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=contoso,DC=com"
```

## 3. User Accounts

Automate the creation of a user account, set its password securely, and immediately assign it to the security group.

```powershell
# Define user variables
$username   = "sysadmin"
$upnSuffix  = "contoso.com"
$securePass = ConvertTo-SecureString "YourPassw0rd!" -AsPlainText -Force
$ouPath     = "OU=IT,DC=contoso,DC=com"

# Create the new enabled user account
New-ADUser -Name $username `
    -SamAccountName $username `
    -UserPrincipalName "$username@$upnSuffix" `
    -Path $ouPath `
    -AccountPassword $securePass `
    -Enabled $true `
    -PasswordNeverExpires $true

# Add the new user to the IT security group
Add-ADGroupMember -Identity "IT" -Members $username
```

## 4. Fine-Grained Password Policies (FGPP)

Create a specific password policy that overrides the default domain policy, and apply it directly to the `IT` security group. This is useful for enforcing stricter requirements for administrative users.

```powershell
# 1. Create the Fine Grained Password Policy (Password Settings Object)
New-ADFineGrainedPasswordPolicy -Name "IT-PSO" `
    -Precedence 10 `
    -ComplexityEnabled $true `
    -MinPasswordLength 16

# 2. Apply the PSO to the IT Security Group
Add-ADFineGrainedPasswordPolicySubject -Identity "IT-PSO" -Subjects "IT"
```

## 5. Verification

Run the following commands to confirm your Active Directory objects were created and configured successfully:

```powershell
# Verify the OU exists
Get-ADOrganizationalUnit -Filter "Name -eq 'IT'"

# Verify the Group exists and check its members
Get-ADGroup -Identity "IT"
Get-ADGroupMember -Identity "IT" | Select-Object Name, objectClass

# Verify the user's creation and UPN
Get-ADUser -Identity "sysadmin" -Properties UserPrincipalName, PasswordNeverExpires

# Verify the FGPP is applied to the correct group
Get-ADFineGrainedPasswordPolicy -Identity "IT-PSO" | Select-Object Name, MinPasswordLength, AppliesTo
```

## 6. Troubleshooting

If you encounter issues during object creation or policy application:

```powershell
# 1. Check if the AD Web Services are running (required for AD PowerShell cmdlets)
Get-Service ADWS

# 2. Verify the effective password policy for a specific user to ensure FGPP is working
Get-ADUserResultantPasswordPolicy -Identity "sysadmin"

# 3. Check for AD replication issues if objects aren't appearing on other Domain Controllers
repadmin /showrepl

# 4. If you accidentally enable "Protect from accidental deletion" and need to remove the OU later:
Set-ADOrganizationalUnit -Identity "OU=IT,DC=contoso,DC=com" -ProtectedFromAccidentalDeletion $false
```

<!-- Created by: Gergő Téringer, 2026 -->