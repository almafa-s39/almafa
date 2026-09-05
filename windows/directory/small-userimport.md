# Low budget User import script

```powershell
$csv = Import-Csv -Path C:\Users\Administrator\Documents\users.csv

foreach ($user in $csv) {
    $ou = Get-ADOrganizationalUnit -Filter "Name -eq '$($user.department)'" -ErrorAction SilentlyContinue

    if (!$ou) {
        New-ADOrganizationalUnit -Path "OU=Users,OU=WSC26,DC=win,DC=learn" -Name $user.department -PassThru
    }

    $group = Get-ADGroup -Filter "Name -eq '$($user.department)'" -ErrorAction SilentlyContinue

    if (!$group) {
        $group = New-ADGroup -Name $user.department -GroupScope Universal -GroupCategory Security -Path "OU=Groups,OU=WSC26,DC=win,DC=learn" -PassThru
    }

    $account = Get-ADUser -Filter "SamAccountName -eq '$($user.sam)'" -ErrorAction SilentlyContinue

    # sam,department,password,sn,givenName

    if (!$account) {
        $pw = ConvertTo-SecureString -AsPlainText -Force $user.password
        $account = New-ADUser -Name "$($user.givenName) $($user.sn)" `
        -DisplayName "$($user.givenName) $($user.sn)" `
        -SamAccountName $user.sam `
        -AccountPassword $pw `
        -Department $user.department `
        -Path "OU=$($user.department),OU=Users,OU=WSC26,DC=win,DC=learn" `
        -Enabled:$true -GivenName `
        $user.givenName -Surname $user.sn `
        -PasswordNeverExpires:$true `
        -PassThru

        Add-ADGroupMember -Members $account -Identity $group
    }
}
```
