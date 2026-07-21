<!-- 
---
title: "Bitlocker"
author: "Gergő Téringer"
---
 -->
# Bitlocker

BitLocker Drive Encryption is a native data protection feature in Windows 11 and Windows Server 2022/2025 that integrates with a Trusted Platform Module (TPM 2.0) to safeguard operating system volumes, fixed data drives, and removable drives against unauthorized offline data access. This guide details enabling BitLocker on the system drive and configuring automatic unlocking for secondary data drives.

## 1. Feature Installation

Depending on whether you are configuring a client operating system (Windows 10/11) or a server operating system (Windows Server 2022/2025), install the BitLocker feature and its management utilities using the appropriate cmdlet.

```powershell
# For Windows Client OS (Windows 11 / Windows 10)
Enable-WindowsOptionalFeature -Online -FeatureName BitLocker, BitLocker-Utilities -All

# For Windows Server OS (Windows Server 2022 / Server 2025)
Install-WindowsFeature -Name BitLocker -IncludeManagementTools -Restart

# Reboot the system if a restart was not automatically triggered
Restart-Computer
```

**Command Breakdown & Explanation:**

- `Enable-WindowsOptionalFeature`: Enables client-side optional Windows features online without requiring offline image servicing.
- `Install-WindowsFeature -Name BitLocker`: Installs the core BitLocker driver and service stack on Windows Server hosts.
- `-IncludeManagementTools`: Installs the BitLocker PowerShell module and GUI management console.
- `Restart-Computer`: Reboots the system to finalize the BitLocker driver initialization.

> [!NOTE]
> Modern operating systems like Windows 11 and Windows Server 2025 strictly require TPM 2.0 and UEFI with Secure Boot enabled to utilize hardware-based BitLocker encryption without additional password/key file overrides.

## 2. System Drive Encryption

Once the feature is installed and the system has rebooted, enable BitLocker on the operating system drive (C:) utilizing the onboard TPM chip as the key protector.

```powershell
Enable-BitLocker -MountPoint "C:" -TpmProtector
```

**Command Breakdown & Explanation:**

- `Enable-BitLocker`: Initiates the encryption process on the specified drive volume.
- `-MountPoint "C:"`: Specifies the operating system volume drive letter to encrypt.
- `-TpmProtector`: Specifies that the primary volume encryption key is secured by the hardware TPM chip.

> [!WARNING]
> In enterprise Active Directory or Microsoft Entra ID environments, always back up the BitLocker Recovery Password to Active Directory (`-RecoveryPasswordProtector`) before locking the volume. Relying solely on `-TpmProtector` without a recovery key backup can result in total data loss if motherboard hardware or firmware is modified.

## 3. Data Drive Auto-Unlock Configuration

Fixed data volumes (such as secondary drive D:) can be configured to automatically unlock whenever the operating system drive (C:) is successfully unlocked by BitLocker during startup.

```powershell
Enable-BitLockerAutoUnlock -MountPoint "D:"
```

**Command Breakdown & Explanation:**

- `Enable-BitLockerAutoUnlock`: Configures a secondary fixed data drive to automatically decrypt upon system startup.
- `-MountPoint "D:"`: Specifies the target secondary drive volume.

>[!IMPORTANT]
> Before enabling `Enable-BitLockerAutoUnlock` on drive D:, the drive must already be encrypted with BitLocker and the operating system drive (C:) must be encrypted and protected by BitLocker.

## 4. Troubleshooting and Verification

Verifying BitLocker deployment involves checking the encryption progress, protection status, and key protectors for each volume.

### 4.1 Verifying Volume Encryption and Protection Status

**Command:** `Get-BitLockerVolume -MountPoint "C:", "D:"`

**What it checks and variables to look for:**

- **MountPoint**: Displays the target volume drive letter (e.g., `C:` or `D:`).
- **ProtectionStatus**: Must show `On` to indicate that volume encryption is active and keys are secured.
- **VolumeStatus**: Displays `FullyEncrypted` when background volume encryption completes. If it shows `EncryptionInProgress`, wait for background processing to finish.
- **LockStatus**: Displays `Unlocked` for active volumes currently accessible by the OS.
- **AutoUnlockEnabled**: Must display `True` on non-system volumes (e.g., `D:`) if automatic unlocking is configured.

### 4.2 Verifying Key Protectors

**Command:** `(Get-BitLockerVolume -MountPoint "C:").KeyProtector`

**What it checks and variables to look for:**

- **KeyProtectorType**: Confirms the active key protection mechanisms attached to the volume (e.g., `Tpm`, `RecoveryPassword`, or `AutoUnlock`).
- **RecoveryPassword**: Confirms whether a 48-digit numerical recovery password exists for emergency access.

<!-- Created by: Gergő Téringer, 2026 -->