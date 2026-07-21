<!-- 
---
title: "Backup ADDS and IIS"
author: "Gergő Téringer"
---
-->
# Backup ADDS and IIS

Automating disaster recovery backups for Active Directory Domain Services (AD DS) components and web infrastructure ensures rapid recovery during server failures. This script consolidates Active Directory user exports, Group Policy Object (GPO) backups, and Internet Information Services (IIS) web root directory archives into a unified pipeline, offloading the final output to a secondary storage target (e.g., an iSCSI drive) and reporting success or failure via SMTP.

```powershell
# Variables
# Emails settings
$smtpServer = "mail.nordicbackup.net"
$from = "backup@skillsnet.dk"
$to = "support@nordicbackup.net"
$subject = ""
$body = ""
$success = $true

# Backup path
$BackupRoot = "C:\Backups"
$usersCSV = "$backupRoot\Users.csv"
$gpoBackup = "$backupRoot\GPOs"
$webBackup = "$backupRoot\Web"

try {
    # Create backup folders
    Write-Output "Create backup folders"
    New-Item -Path $BackupRoot -ItemType Directory -Force | Out-Null
    New-Item -Path $gpoBackup -ItemType Directory -Force | Out-Null
    New-Item -Path $webBackup -ItemType Directory -Force | Out-Null

    # Export users to CSV
    # FirstName,LastName,samAccountName,UserPrincipalName,Email,JobTitle,City,Company,Department
    Write-Output "Export users to CSV"
    Get-ADUser -Filter * -Properties GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Select-Object GivenName, Surname, SamAccountName, DistinguishedName, UserPrincipalName, EmailAddress, Title, City, Company, Department `
        | Export-Csv -Path $usersCSV -NoTypeInformation -Encoding UTF8


    # Export Group Policy Objects
    Write-Output "Export Group Policy Objects"
    $allGPO = Get-GPO -All
    foreach ($gpo in $allGPO) {
        $gpoName = $gpo.DisplayName
        $gpoPath = Join-Path $gpoBackup $gpoName
        New-Item -Path $gpoPath -ItemType Directory -Force | Out-Null
        Backup-GPO -Name $gpoName -Path $gpoPath -ErrorAction SilentlyContinue
    }

    # Backup IIS web root folders
    Write-Output "Backup IIS web root folders"
    Import-Module WebAdministration

    # Get all IIS sites
    $sites = Get-Website

    foreach ($site in $sites) {
        $siteName = $site.Name
    
        $sourcePath = $site.PhysicalPath
        $sourcePath = $sourcePath.Replace('%SystemDrive%', 'C:')
        $destinationPath = "C:\Backups\Web\$siteName"

        Write-Host "Backing up site '$siteName' from $sourcePath to $destinationPath"

        # Create destination folder
        New-Item -ItemType Directory -Path $destinationPath -Force | Out-Null

        # Copy the site files
        Copy-Item -Path $sourcePath\ -Destination $destinationPath -Recurse -Force -ErrorAction Stop
    }

    
    # Copy items to iSCSI disk
    Copy-Item -Path $BackupRoot -Destination "b:\" -Recurse -Force
    
    # Send success email
    Write-Output "Sending success email"
    $subject = "Backup Success on $env:COMPUTERNAME"
    $body = "The backup completed successfully on $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')."
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
} catch {
    # Sending failure email
    Write-Output "Sending failure email"
    $succes = $false
    $subject = "Backup FAILED on $env:COMPUTERNAME"
    $body = "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
    Send-MailMessage -From $from -To $to -Subject $subject -Body $body -SmtpServer $smtpServer
    Write-Output "Backup failed on $(Get-Date). Error: $($_.Exception.Message)"
}
```

**Command Breakdown & Explanation:**

- `try { ... } catch { ... }`: Encapsulates the entire workflow in an exception handling block. Any terminating error (such as a missing source path or inaccessible drive) immediately stops execution and jumps to the `catch` block to send a failure alert.
- `Get-ADUser -Filter * -Properties ... | Export-Csv`: Queries Active Directory for all user accounts and retrieves extended attributes. `Export-Csv -NoTypeInformation -Encoding UTF8` exports the dataset to a structured file without headers describing the object type.
- `Get-GPO -All` & `Backup-GPO`: Enumerates every Group Policy Object in the domain and exports the policy settings, XML files, and security descriptors into individual subdirectories.
- `Import-Module WebAdministration` & `Get-Website`: Loads the IIS PowerShell management module to query running websites and dynamically resolve their physical file paths on disk.
- `$sourcePath.Replace('%SystemDrive%', 'C:')`: Replaces IIS environment variables with explicit drive letters to prevent file copy path errors.
- `Send-MailMessage`: Connects to the designated SMTP server to transmit automated execution status reports containing system hostnames and timestamped error strings.

<!-- Created by: Gergő Téringer, 2026 -->