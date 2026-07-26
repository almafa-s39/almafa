# Fine-Grained Password Policies in Windows Server 2025

Fine-Grained Password Policies (FGPP) let you apply different password
and account-lockout rules to different sets of users within a single
Active Directory domain — for example, stricter settings for privileged
accounts than for regular users — without needing multiple domains.

This document covers creating, viewing, editing, and deleting FGPPs on
Windows Server 2025 using both the Active Directory Administrative
Center (ADAC) GUI and the Active Directory PowerShell module.

> [!NOTE]
> A Fine-Grained Password Policy is stored as a Password Settings Object
> (PSO) inside Active Directory. PSOs can only be applied directly to
> **user objects** and **global security groups** — they cannot be
> linked to an Organizational Unit (OU) the way Group Policy Objects
> can.

## 1. Prerequisites

Before creating fine-grained password policies, confirm the following:

- The domain functional level must be Windows Server 2012 or higher.
- You must be a member of the **Domain Admins** group (or have been
  delegated equivalent rights over the Password Settings Container).
- Either the Active Directory Administrative Center (ADAC) or the
  Active Directory module for Windows PowerShell must be installed —
  both are part of the Remote Server Administration Tools (RSAT) /
  AD DS role administration tools.

```powershell
Get-WindowsFeature RSAT-AD-AdminCenter, RSAT-AD-PowerShell
```

What it checks and variables to look for:

- **Install State**: Must be `Installed` for both features listed. If
  either shows `Available`, install it with `Install-WindowsFeature
  RSAT-AD-AdminCenter, RSAT-AD-PowerShell` before continuing.

## 2. Creating a Fine-Grained Password Policy

### 2.1 GUI Method (Active Directory Administrative Center)

1. Open **Active Directory Administrative Center** — from the **Tools**
   menu of Server Manager, or by running `dsac.exe` in an elevated
   PowerShell session.
2. If the target domain isn't shown, choose **Manage** → **Add
   Navigation Nodes**, select the domain, and choose **OK**.
3. In the navigation pane, expand **System**, then select **Password
   Settings Container**.
4. In the **Tasks** pane, choose **New** → **Password Settings**.
5. Fill in the property page. **Name** and **Precedence** are required;
   set the remaining password and lockout values as needed.
6. Under **Directly Applies To**, choose **Add**, type the name of the
   user or global security group the policy should apply to, and
   choose **OK**.
7. Choose **OK** to create the policy.

> [!TIP]
> Lower **Precedence** numbers win when a user is a member of groups
> covered by more than one PSO. Plan your precedence numbering (e.g.,
> `10` for the most privileged tier, `500` for a general baseline) before
> creating multiple policies.

### 2.2 PowerShell Method

```powershell
New-ADFineGrainedPasswordPolicy -Name "AdminTierPSO" `
    -Precedence 10 `
    -ComplexityEnabled $true `
    -MinPasswordLength 16 `
    -PasswordHistoryCount 24 `
    -MaxPasswordAge "60.00:00:00" `
    -MinPasswordAge "1.00:00:00" `
    -LockoutThreshold 5 `
    -LockoutDuration "00:30:00" `
    -LockoutObservationWindow "00:30:00" `
    -ReversibleEncryptionEnabled $false
```

**Command Breakdown & Explanation:**

- `New-ADFineGrainedPasswordPolicy`: Creates a new PSO in the Password
  Settings Container.
- `-Name "AdminTierPSO"`: The PSO's name (also used as its `cn`
  attribute); must be unique in the container.
- `-Precedence 10`: A lower value takes priority over PSOs with higher
  values when a subject is covered by more than one policy. This
  parameter is mandatory.
- `-ComplexityEnabled $true`: Enforces Windows password complexity
  requirements (mixed case, digits, symbols).
- `-MinPasswordLength 16`: Minimum number of characters required.
- `-PasswordHistoryCount 24`: Number of previous passwords remembered to
  prevent reuse.
- `-MaxPasswordAge "60.00:00:00"`: Maximum password age, expressed as a
  `TimeSpan` string (`days.hh:mm:ss`) — 60 days here.
- `-MinPasswordAge "1.00:00:00"`: Minimum time before a password can be
  changed again — 1 day here, which discourages rapid history cycling.
- `-LockoutThreshold 5`: Number of failed logon attempts before the
  account locks.
- `-LockoutDuration "00:30:00"`: How long the account stays locked — 30
  minutes here; a value of `0` requires manual admin unlock.
- `-LockoutObservationWindow "00:30:00"`: The time window in which
  failed attempts are counted toward the lockout threshold.
- `-ReversibleEncryptionEnabled $false`: Keeps passwords stored using
  one-way hashing rather than reversible encryption; leave this `$false`
  unless a specific application requires the reversible format.

> [!WARNING]
> `-LockoutObservationWindow` cannot exceed `-LockoutDuration`. Setting
> an invalid combination causes `New-ADFineGrainedPasswordPolicy` to
> fail with a constraint violation error.

## 3. Applying the Policy to Users or Groups

### 3.1 GUI Method

Subjects are added directly during policy creation via **Directly
Applies To** → **Add** (Section 2.1, step 6). To add or remove subjects
later, edit the policy's properties (see Section 5.1) and modify the
**Directly Applies To** list.

### 3.2 PowerShell Method

```powershell
Add-ADFineGrainedPasswordPolicySubject -Identity "AdminTierPSO" -Subjects "Tier0-Admins"
```

**Command Breakdown & Explanation:**

- `-Identity "AdminTierPSO"`: The PSO to modify, identified by name,
  distinguished name, or GUID.
- `-Subjects "Tier0-Admins"`: One or more users or **global security
  groups** (not OUs) to which the policy now applies. Multiple subjects
  can be supplied as a comma-separated list.

> [!IMPORTANT]
> Since PSOs cannot target an OU directly, the common pattern is to
> create a dedicated global security group (a "shadow group") that
> mirrors the membership of the OU you actually want covered, and apply
> the PSO to that group instead.

```powershell
Get-ADFineGrainedPasswordPolicySubject -Identity "AdminTierPSO"
```

Lists every user and group the policy is currently applied to — useful
for auditing before making further changes.

```powershell
Remove-ADFineGrainedPasswordPolicySubject -Identity "AdminTierPSO" -Subjects "jsmith"
```

Removes a single subject (`jsmith`) from the policy without deleting the
PSO itself or affecting its other subjects.

## 4. Viewing the Resultant Policy for a User

Because a user could theoretically be covered by more than one PSO
through different group memberships, Active Directory calculates a
single winning policy per user: the Resultant Set of Policy (RSoP),
based on `msDS-ResultantPSO` and the lowest precedence value among all
applicable PSOs.

### 4.1 GUI Method

```markdown
1. In ADAC, navigate to the user account in question.
2. In the **Tasks** pane, choose **View Resultant Password Settings**.
3. Review the effective values, then choose **Cancel** (this view is
   read-only).
```

### 4.2 PowerShell Method

```powershell
Get-ADUserResultantPasswordPolicy -Identity "jsmith"
```

**Command Breakdown & Explanation:**

- `Get-ADUserResultantPasswordPolicy`: Returns the single PSO object
  that is actually in effect for the specified user, resolving
  precedence automatically. Returns nothing if no FGPP applies, meaning
  the domain's default password policy governs that user instead.

## 5. Editing a Fine-Grained Password Policy

### 5.1 GUI Method

1. In ADAC, expand **System** → **Password Settings Container**.
2. Select the policy to edit, and choose **Properties** in the **Tasks**
   pane.
3. Modify the desired settings and choose **OK**.

### 5.2 PowerShell Method

```powershell
Set-ADFineGrainedPasswordPolicy -Identity "AdminTierPSO" -PasswordHistoryCount 30
```

**Command Breakdown & Explanation:**

- `Set-ADFineGrainedPasswordPolicy`: Modifies an existing PSO in place.
- `-Identity "AdminTierPSO"`: The PSO to modify.
- `-PasswordHistoryCount 30`: Example of changing a single setting;
  any parameter accepted by `New-ADFineGrainedPasswordPolicy` can be
  updated the same way (`-Precedence`, `-MinPasswordLength`,
  `-LockoutThreshold`, etc.).

## 6. Deleting a Fine-Grained Password Policy

PSOs created through ADAC are protected from accidental deletion by
default, so removal is a two-step process: clear the protection flag,
then delete the object.

### 6.1 GUI Method

1. In ADAC, expand **System** → **Password Settings Container**.
2. Select the policy, choose **Properties**, clear the **Protect from
   accidental deletion** checkbox, and choose **OK**.
3. Select the policy again, choose **Delete** in the **Tasks** pane, and
   confirm in the dialog.

### 6.2 PowerShell Method

```powershell
Set-ADFineGrainedPasswordPolicy -Identity "AdminTierPSO" -ProtectedFromAccidentalDeletion $false
Remove-ADFineGrainedPasswordPolicy -Identity "AdminTierPSO"
```

**Command Breakdown & Explanation:**

- `-ProtectedFromAccidentalDeletion $false`: Must be cleared first if
  the object still has deletion protection enabled, or the removal
  command below will fail.
- `Remove-ADFineGrainedPasswordPolicy -Identity "AdminTierPSO"`: Deletes
  the PSO. This does not affect the users/groups it was applied to —
  they simply fall back to the next-highest-precedence PSO, or the
  domain default policy if none remain.

> [!CAUTION]
> `Get-ADFineGrainedPasswordPolicy -Filter {Name -like "*test*"} |
> Remove-ADFineGrainedPasswordPolicy` will bulk-delete every matching
> PSO. Always run the `Get-ADFineGrainedPasswordPolicy -Filter ...` half
> alone first to confirm exactly which objects match before piping into
> `Remove-`.

## 7. Verification and Troubleshooting

### 7.1 Verify a policy exists and inspect its settings

```powershell
Get-ADFineGrainedPasswordPolicy -Identity "AdminTierPSO" -Properties *
```

What it checks and variables to look for:

- **Precedence**: Confirm it matches your intended priority ordering
  relative to other PSOs.
- **msDS-PSOAppliesTo**: Should list the distinguished name(s) of the
  intended users/groups.
- **AppliesTo**: The friendlier, resolved equivalent of
  `msDS-PSOAppliesTo` shown by default without needing `-Properties *`.

### 7.2 List all fine-grained password policies in the domain

```powershell
Get-ADFineGrainedPasswordPolicy -Filter *
```

What it checks and variables to look for:

- **Name / Precedence**: Review the full list to check for precedence
  collisions (two PSOs with the same number produce an
  administrator-defined, effectively unpredictable tie-break) or
  unintended overlaps between policies.

### 7.3 Confirm the effective policy for a specific user

```powershell
Get-ADUserResultantPasswordPolicy -Identity "jsmith"
```

What it checks and variables to look for:

- **Output present vs. empty**: A returned object confirms an FGPP is
  in effect for the user; no output means the domain default password
  policy applies instead — check that this matches your expectation.
- **Name**: Confirms *which* PSO won, useful when a user could
  potentially be covered by multiple overlapping policies via different
  group memberships.

### 7.4 Confirm domain functional level supports FGPP

```powershell
Get-ADDomain | Select-Object DomainMode
```

What it checks and variables to look for:

- **DomainMode**: Must be `Windows2012Domain` or a higher-numbered mode.
  If it shows an older mode, fine-grained password policies cannot be
  created until the domain functional level is raised.

### 7.5 Common issues

- `New-ADFineGrainedPasswordPolicy` fails with an attribute constraint
  error: Check that `-LockoutObservationWindow` does not exceed
  `-LockoutDuration`, and that numeric values (length, history count)
  are within Active Directory's allowed ranges.
- A PSO exists but doesn't seem to apply to a user: Confirm the subject
  is a **global security group** or user object, not an OU or a
  domain-local/universal group — PSOs silently ignore unsupported
  principal types in `-Subjects`.
- `Remove-ADFineGrainedPasswordPolicy` fails with an "access is denied"
  or protection-related error: The object still has
  `ProtectedFromAccidentalDeletion` set to `$true`; clear it first (see
  Section 6.2).
- Two policies both seem plausible for a user and the wrong one wins:
  Use `Get-ADUserResultantPasswordPolicy` (Section 7.3) rather than
  manually reasoning about precedence — it resolves ties and
  cross-group overlaps authoritatively.
