<!-- 
---
title: "PowerShell AD Import and User Creation Management"
author: "Gergő Téringer"
---
-->
# PowerShell AD Import and User Creation Management

Automating Active Directory (AD) user and group management is a critical skill for system administrators. The following scripts demonstrate how to rapidly provision test environments using bulk creation loops, as well as how to perform structured imports from a CSV file while dynamically building the necessary Organizational Unit (OU) hierarchy.

## 1. Bulk User Creation Script

This script takes a `-count <n>` argument and creates **n** users in all of the groups specified in the script. The OU and the security groups have to be created in advance.

```powershell
Param(
    [int] $count
)

$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd"
$longpassword = ConvertTo-SecureString -AsPlainText -Force "Passw0rd@Passw0rd"

$basepath = ",dc=paris,dc=local"

$groups = @(
    @{username='mkt';ou='MKT';password=$password;group="cn=MKT,ou=MKT$($basepath)"},
    @{username='sales';ou='SALES';password=$password;group="cn=SALES,ou=SALES$($basepath)"},
    @{username='tech';ou='TECH';password=$password;group="cn=TECH,ou=TECH$($basepath)"},
    @{username='hr';ou='HR';password=$longpassword;group="cn=HR,ou=HR$($basepath)"}
)

foreach ($group in $groups) {
    for ($i = 1; $i -le $count; $i++) {
        $username = "$($group.username)$($i)"
        
        if (Get-ADUser -Filter {SamAccountName -eq $username}) {
            Write-Host "Skipping user $($username)"
            continue
        }

        Write-Host "Creating user $($username)"
        
        $user = New-ADUser -Name $username `
            -SamAccountName $username `
            -PassThru `
            -Path "ou=$($group.ou)$($basepath)" `
            -PasswordNeverExpires $true `
            -ChangePasswordAtLogon $false `
            -AccountPassword $group.password `
            -Enabled $true

        Add-ADGroupMember -Identity $group.group -Members $user
    }
}
```

For example, you can call this script like this:

```powershell
./create_user.ps1 -count 5
```

This way, the script will create 5 users in each group.

**Command Breakdown & Explanation:**

- `Param( [int] $count )`: Defines the script parameter, allowing you to pass the target number of users at runtime (e.g., `./create_user.ps1 -count 5`).
- `ConvertTo-SecureString ... -AsPlainText -Force`: Active Directory cmdlets require passwords to be passed as SecureString objects rather than plain text. This command performs that necessary conversion.
- `$groups = @(...)`: Creates an array of hash tables. Each hash table acts as a configuration template for a specific department, storing the username prefix, target OU, required password complexity, and the Distinguished Name (DN) of the security group.
- `Get-ADUser -Filter ...`: Queries the directory before attempting creation. If the user already exists, it skips to the next iteration to prevent terminating errors.
- `New-ADUser -PassThru`: The `-PassThru` switch is highly useful here. By default, `New-ADUser` does not return any output. `-PassThru` forces it to output the newly created user object, which is then captured in the `$user` variable for the next step.
- `Add-ADGroupMember`: Takes the `$user` object captured in the previous step and adds them to the security group defined in the department's hash table.

## 2. AD DS User Import from CSV

```powershell
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Skills
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Users -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Groups -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Computers -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Servers -Path "OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Employees -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Tech -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Sales -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Finance -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Development -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"
New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $false -Name Constractors -Path "OU=Users,OU=Skills,DC=skillsnet,DC=dk"

#ATTRIBUTES Firstname,LastName,samAccountName,userPrincipalName,Email,JobTitle,City,Company,Department
$path = "c:\users.csv"
$password = ConvertTo-SecureString -AsPlainText -Force "Passw0rd!"
$csv = Import-Csv -Path $path
$i = 1
foreach ($user in $csv) {
  $finame = $user.FirstName
  $lname = $user.LastName
  $funame = $finame + " " + $lname
  $sam = $finame + "." + $lname
  $pname = $user.UserPrincipalName
  $email = $user.Email
  $job = $user.JobTitle
  $city = $user.City
  $company = $user.Company
  $dep = $user.Department
  
  $exUser = Get-ADUser -Filter (SamAccountName -eq $sam) -ErrorAction SilentlyContinue
  $exGrp = Get-ADGroup -Filter (Name -eq $dep) -SearchBase "OU=Groups,OU=Skills,DC=skillsnet,DC=dk" -ErrorAction SilentlyContinue
  
  if (!$exGrp) {
    New-ADGroup -Name $dep -Path "OU=Groups,OU=Skills,DC=skillsnet,DC=dk" -GroupScope Global
  }
  
  if ($exUser) {
    Write-Host "$i. The $sam user exists..."
  } else {
    New-AdUser `
      -Name $funame `
      -GivenName $finame `
      -Surname $lname `
      -SamAccountName $sam `
      -UserPrincipalName $pname `
      -Path "OU=$dep,OU=Users,DC=skillsnet,DC=dk" `
      -Title $job `
      -AccountPassword $password `
      -Enabled $true `
      -Department $dep `
      -City $city `
      -Company $company ` 
      -EmailAddress $email `
      -DisplayName $funame
      
    Add-AdGroupMembership -Identity $dep -Members $sam
      
    Write-Host "$i. The $sam has been successfully created."
    $i++
  }
}
```

**Command Breakdown & Explanation:**

- `New-ADOrganizationalUnit`: Builds the directory tree. The `-ProtectedFromAccidentalDeletion $false` parameter is explicitly defined here, which is standard practice in lab or testing environments to allow for easy teardown later.
- `Import-Csv`: Reads the specified file and parses the top row as headers. This allows you to call specific column data inside the loop using dot notation (e.g., `$user.FirstName`).
- `Get-ADGroup ... -ErrorAction SilentlyContinue`: Searches the defined `Groups` OU to see if a security group matching the user's `$dep` (Department) exists. `SilentlyContinue` suppresses the red error text if the group is not found, allowing the script logic to cleanly proceed to the `if (!$exGrp)` creation block.
- `New-ADGroup -GroupScope Global`: Dynamically generates the missing department group before the user creation phase so the user has a valid target to join.
- `Add-AdGroupMembership -Members $sam`: Maps the newly created user to their departmental group. The AD cmdlets natively resolve the `SamAccountName` string variable provided here into the actual user object behind the scenes.

<!-- Created by: Gergő Téringer, 2026 -->