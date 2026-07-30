<!-- 
---
title: "SMB"
author: "Gergő Téringer"
---
 -->
# SMB

This document provides administrative procedures for configuring global Server Message Block (SMB) server settings and managing individual network file shares. The commands and configurations described target modern Windows environments, such as Windows 11 and Windows Server 2025, ensuring transport-level security, high performance, and role-based access control.

> [!NOTE]
> Although frequently referred to under the generic term "Samba" in heterogeneous IT environments, `Set-SmbServerConfiguration` and `New-SmbShare` are native Microsoft PowerShell cmdlets provided by the `SmbShare` module.

## 1. Global SMB Server Configuration

> [!IMPORTANT]
> Enforcing server-wide SMB encryption (`-EncryptData:$true` and `-RejectUnencryptedAccess:$true`) prevents network eavesdropping and man-in-the-middle (MitM) attacks. Legacy SMB clients that do not support SMB 3.x encryption will be denied access automatically.

```powershell
Set-SmbServerConfiguration -EncryptData:$true -MaxChannelPerSession 16 -RejectUnencryptedAccess:$true -Force
```

**Command Breakdown & Explanation:**

- `-EncryptData:$true`: Mandates end-to-end SMB 3.x transport encryption across all shares hosted by the server.
- `-MaxChannelPerSession 16`: Configures SMB Multichannel to permit up to 16 parallel connections per session, maximizing throughput across multi-NIC or RSS-enabled (Receive Side Scaling) interfaces.
- `-RejectUnencryptedAccess:$true`: Blocks any client connection request that does not support or request transport encryption.
- `-Force`: Suppresses interactive confirmation prompts during automated deployment.

## 2. SMB Share Management

> [!TIP]
> Share permissions control network access, whereas NTFS/ReFS File System ACLs control granular directory and file access. Apply the Principle of Least Privilege across both access layers.

```powershell
# Provision a new SMB share with explicit Active Directory permissions and share-level encryption
New-SmbShare -Name "FinanceData" `
             -Path "C:\Shares\FinanceData" `
             -ReadAccess "DOMAIN\Finance-Readers" `
             -ChangeAccess "DOMAIN\Finance-Editors" `
             -FullAccess "DOMAIN\Finance-Admins" `
             -NoAccess "DOMAIN\Contractors" `
             -EncryptData $true `
             -FolderEnumerationMode AccessBased

# Update an existing SMB share to enforce encryption
Set-SmbShare -Name "FinanceData" `
             -EncryptData $true `
             -Force
```

**Command Breakdown & Explanation:**

- `New-SmbShare` / `Set-SmbShare`: PowerShell cmdlets used to create new network shares or modify existing share properties.
- `-Name "FinanceData"`: Defines the visible Universal Naming Convention (UNC) share identifier (e.g., `\\ServerName\FinanceData`).
- `-Path "C:\Shares\FinanceData"`: Specifies the local physical folder path hosting the share content.
- `-ReadAccess "DOMAIN\Finance-Readers"`: Grants read-only network share permissions to the designated Active Directory user or group.
- `-ChangeAccess "DOMAIN\Finance-Editors"`: Grants read, write, execute, and delete share permissions.
- `-FullAccess "DOMAIN\Finance-Admins"`: Grants full administrative share permissions, including permission modification rights.
- `-NoAccess "DOMAIN\Contractors"`: Explicitly denies share access, overriding any conflicting inherited group memberships.
- `-EncryptData $true`: Enforces SMB encryption specifically for this share, overriding server defaults if global encryption is set to optional.
- `-FolderEnumerationMode AccessBased`: If a user don't have permission to the folder, it is not listed. By default it is disabled (`Unrestricted`).

## 3. Verification and Troubleshooting

> [!NOTE]
> Execute these verification cmdlets in an elevated PowerShell terminal on Windows 11 or Windows Server 2025 to validate global SMB policies, share parameters, and active connection security.

### 3.1 Verify global SMB server security settings

**Command:** `Get-SmbServerConfiguration | Select-Object EncryptData, RejectUnencryptedAccess, MaxChannelPerSession`

**What it checks and variables to look for:**

- **EncryptData**: Must be `True`
- **RejectUnencryptedAccess**: Must be `True`
- **MaxChannelPerSession**: Must be `16`

### 3.2 Verify individual SMB share properties and encryption flags

**Command:** `Get-SmbShare -Name "FinanceData" | Select-Object Name, Path, EncryptData`

**What it checks and variables to look for:**

- **Name**: Must be `FinanceData`
- **Path**: Must be `C:\Shares\FinanceData`
- **EncryptData**: Must be `True`

### 3.3 Verify active client sessions, SMB dialects, and encryption enforcement

**Command:** `Get-SmbSession | Select-Object ClientComputerName, ClientUserName, Dialect, Encrypted`

**What it checks and variables to look for:**

- **Dialect**: Must be `3.0`, `3.02`, or `3.1.1`
- **Encrypted**: Must be `True`

<!-- Created by: Gergő Téringer, 2026 -->