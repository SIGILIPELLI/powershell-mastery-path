# 08 · Active Directory & System Admin Cmdlets

!!! warning "Manual review only — not executed"
    This machine has no Windows domain controller and no `ActiveDirectory`
    module (it's Windows-only and requires RSAT or a domain-joined
    server), so none of the code in this module could be run for real. It
    has been reviewed carefully by hand against the documented behavior of
    the `ActiveDirectory` module's cmdlets, and the shapes of the commands
    and their parameters are accurate — but no output below comes from an
    actual execution, and you should treat this module as reference
    material to validate against a real AD environment before relying on
    it in production. This is the one module in the course where that
    caveat applies; every other module's code was executed for real.

Active Directory administration is one of PowerShell's original reasons
for existing — most of what a Windows AD admin does interactively in
Active Directory Users and Computers has a scriptable equivalent.

## Prerequisites

```powershell
Install-WindowsFeature -Name RSAT-AD-PowerShell   # on a Windows Server
# or, on Windows client:
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0

Import-Module ActiveDirectory
Get-Command -Module ActiveDirectory | Measure-Object   # hundreds of cmdlets
```

Every cmdlet in the module implicitly targets the domain your session is
authenticated to unless you pass `-Server` — worth being deliberate about
in a multi-domain forest.

## Querying users

```powershell
Get-ADUser -Filter "Department -eq 'Engineering'" -Properties Title, Mail |
    Select-Object Name, Title, Mail
```

```text
Name        Title              Mail
----        -----              ----
Jane Doe    Senior Engineer    jane.doe@corp.example.com
Sam Patel   Engineer           sam.patel@corp.example.com
```

`-Filter` uses PowerShell-style syntax (not raw LDAP), which is why it
looks like a normal comparison expression. Note `-Properties` is required
to get anything beyond the small default attribute set — `Get-ADUser`
without it won't return `Title` or `Mail` even if you ask for them in
`Select-Object`; the property has to actually be fetched from AD first.

## The trap: `-Filter` string syntax vs `-LDAPFilter`

```powershell
# Works: PowerShell-style filter, evaluated by the cmdlet
Get-ADUser -Filter "SamAccountName -eq 'jdoe'"

# Also works, but a different dialect entirely - raw LDAP syntax
Get-ADUser -LDAPFilter "(sAMAccountName=jdoe)"

# Common mistake: mixing the two syntaxes in one string silently returns nothing or errors
Get-ADUser -Filter "(sAMAccountName=jdoe)"   # wrong: LDAP syntax inside -Filter
```

`-Filter` and `-LDAPFilter` are two separate parameters with two separate
grammars — copying an LDAP query you found somewhere into `-Filter`
verbatim is one of the most common AD scripting mistakes, and it fails
quietly (empty results) rather than with an obvious syntax error in most
cases.

## Creating and modifying users

```powershell
$password = ConvertTo-SecureString "TempP@ss123!" -AsPlainText -Force

New-ADUser -Name "Alex Kim" `
    -SamAccountName "akim" `
    -UserPrincipalName "akim@corp.example.com" `
    -Path "OU=Engineering,DC=corp,DC=example,DC=com" `
    -AccountPassword $password `
    -Enabled $true `
    -ChangePasswordAtLogon $true
```

```powershell
Set-ADUser -Identity "akim" -Title "Software Engineer" -Department "Engineering"
```

```powershell
Set-ADAccountPassword -Identity "akim" -NewPassword $password -Reset
Unlock-ADAccount -Identity "akim"
```

`-Enabled $true` at creation time matters: `New-ADUser` creates *disabled*
accounts by default unless you either set this or provide a password that
satisfies policy — a very common "why can't this new user log in"
support ticket traces back to this exact default.

## Groups and membership

```powershell
Add-ADGroupMember -Identity "Engineering-Team" -Members "akim", "jdoe"
Get-ADGroupMember -Identity "Engineering-Team" | Select-Object Name, SamAccountName
Remove-ADGroupMember -Identity "Engineering-Team" -Members "akim" -Confirm:$false
```

`Remove-ADGroupMember` prompts for confirmation by default (it's a
destructive membership change) — `-Confirm:$false` is what a non-interactive
script needs to run unattended, but only pass it once you're confident the
script's logic upstream is correct, since it removes the safety net entirely.

## Computers and OUs

```powershell
Get-ADComputer -Filter "OperatingSystem -like '*Server*'" -Properties OperatingSystem |
    Select-Object Name, OperatingSystem

New-ADOrganizationalUnit -Name "Contractors" -Path "DC=corp,DC=example,DC=com" `
    -ProtectedFromAccidentalDeletion $true
```

`-ProtectedFromAccidentalDeletion $true` sets AD's own delete-protection
flag on the OU object — a real safeguard worth setting on every OU you
create by script, since a stray `Remove-ADOrganizationalUnit -Recursive`
elsewhere in a larger automation run is exactly the kind of mistake this
flag exists to stop.

## The trap: bulk operations without `-WhatIf` first

```powershell
# Disable every account that hasn't logged in for 180+ days
$cutoff = (Get-Date).AddDays(-180)

Get-ADUser -Filter "Enabled -eq 'True'" -Properties LastLogonDate |
    Where-Object { $_.LastLogonDate -lt $cutoff } |
    Disable-ADAccount -WhatIf
```

Any cmdlet that supports `SupportsShouldProcess` (nearly everything that
changes AD state — `Disable-ADAccount`, `Remove-ADUser`,
`Set-ADAccountPassword`) accepts `-WhatIf`, which prints what *would*
happen without doing it. Running the dry-run version of a bulk filter is
non-negotiable before removing `-WhatIf` and running for real — a filter
typo here (`-lt` vs `-gt`, wrong property) can disable the wrong 5,000
accounts instead of the right 12.

## Cheat sheet

| Cmdlet | Purpose |
|---|---|
| `Get-ADUser -Filter "..." -Properties ...` | query users; `-Properties` needed beyond defaults |
| `-Filter` (PS syntax) vs `-LDAPFilter` (raw LDAP) | different grammars, don't mix |
| `New-ADUser ... -Enabled $true` | create; disabled by default without this |
| `Set-ADUser` / `Set-ADAccountPassword` | modify attributes / reset password |
| `Add-ADGroupMember` / `Remove-ADGroupMember -Confirm:$false` | manage group membership |
| `Get-ADComputer -Filter ...` | query computer objects |
| `New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $true` | create OU with delete protection |
| `-WhatIf` | dry-run any state-changing AD cmdlet before running for real |

## Exercise

On a real (or lab/test) AD environment: write a script that finds all
disabled user accounts in a given OU, exports them to CSV with
`Name`, `SamAccountName`, and `whenChanged`, then — after reviewing the
CSV — offers a `-Confirm`-gated bulk deletion of accounts disabled more
than a year ago. Test the query and export logic against a real
environment before ever running the deletion step, and keep `-WhatIf` on
until you've manually verified the CSV output is exactly the set you
expect.
