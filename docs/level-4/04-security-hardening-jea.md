# 04 · Security Hardening (JEA)

Level 3's security module covered protecting secrets *inside* scripts.
This module covers restricting what a remote user can do *at all* — Just
Enough Administration (JEA) lets you grant someone remote PowerShell
access limited to a specific, named set of commands, running as a
different (often more privileged) account than their own — without ever
handing them full admin rights or a real admin password.

!!! note "What ran for real here"
    `New-PSRoleCapabilityFile` and `New-PSSessionConfigurationFile` are
    cross-platform cmdlets that generate configuration files — both ran
    for real below, with genuine generated output. `Register-PSSessionConfiguration`,
    which actually activates a JEA endpoint over WinRM, is Windows-only
    and wasn't available to run on this machine; that section is
    documented for accuracy but not executed.

## The problem JEA solves

Without JEA, remote PowerShell access is all-or-nothing: `Invoke-Command`
against a session either works with the full authority of whatever
account connects, or it's blocked entirely. A help-desk technician who
only ever needs to restart a stuck service on a server has no
in-between option — either they get a real admin session (far more power
than the task needs), or they don't get PowerShell access at all.

## Step 1: a role capability file — what commands are allowed

```powershell
New-PSRoleCapabilityFile -Path ./ServiceDeskOperator.psrc `
    -VisibleCmdlets 'Restart-Service', 'Get-Service', 'Get-Process' `
    -VisibleFunctions 'Get-DiskSpaceReport' `
    -Description 'Allows restarting services and basic diagnostics only'
```

```text
@{
GUID = '394ea5f3-f932-409c-aba5-bba82dae5b01'
Author = 'bhanuja'
Description = 'Allows restarting services and basic diagnostics only'
...
VisibleCmdlets = 'Restart-Service', 'Get-Service', 'Get-Process'
VisibleFunctions = 'Get-DiskSpaceReport'
```

A `.psrc` file is the allow-list: it names exactly which cmdlets,
functions, and external programs are visible inside a session using this
role — everything not listed simply doesn't exist for that user, not
even to be discovered with `Get-Command`. `-VisibleFunctions` can point at
your own module's functions too (like `Get-DiskSpaceReport` from Level
3's admin toolkit project) — JEA restricts to a curated subset of *your*
tooling just as easily as built-in cmdlets.

You can go further than a flat list with `-VisibleCmdlets` entries that
constrain *parameters*, not just command names:

```powershell
VisibleCmdlets = @{
    Name = 'Restart-Service'
    Parameters = @{ Name = 'Name'; ValidateSet = 'Spooler', 'BITS' }
}
```

That restricts `Restart-Service` to only two named services — the
operator can restart the print spooler or BITS, but not, say, a database
service, even though `Restart-Service` itself is "allowed."

## Step 2: a session configuration file — who gets which role, and as whom

```powershell
New-PSSessionConfigurationFile -Path ./ServiceDeskEndpoint.pssc `
    -SessionType RestrictedRemoteServer `
    -RunAsVirtualAccount `
    -RoleDefinitions @{ 'CORP\ServiceDeskOperators' = @{ RoleCapabilities = 'ServiceDeskOperator' } }
```

```text
SessionType = 'RestrictedRemoteServer'
RunAsVirtualAccount = $true
RoleDefinitions = @{
    'CORP\ServiceDeskOperators' = @{
        'RoleCapabilities' = 'ServiceDeskOperator' } }
```

Three settings doing the real security work:

- **`SessionType RestrictedRemoteServer`** — starts from a minimal
  baseline (no `Invoke-Expression`, heavily curated built-in cmdlet set)
  rather than a full PowerShell session, so anything not explicitly
  granted by a role capability stays unavailable by default.
- **`RunAsVirtualAccount`** — the connecting user's own identity is never
  what executes the commands; a temporary, auto-managed local admin
  account runs them instead. The help-desk technician's own domain
  credentials never need local admin rights on the target machine at all
  — only the virtual account does, and it exists only for the session.
- **`RoleDefinitions`** — maps a security group (`CORP\ServiceDeskOperators`)
  to the role capability file that applies to it. Different groups can
  map to different `.psrc` files on the same endpoint, layering multiple
  restricted roles behind one connection point.

## Step 3: registering the endpoint (Windows, WinRM)

```powershell
Register-PSSessionConfiguration -Name "ServiceDesk" `
    -Path ./ServiceDeskEndpoint.pssc -Force
```

This registers the endpoint with WinRM so `Invoke-Command -ConfigurationName
ServiceDesk` (or `Enter-PSSession -ConfigurationName ServiceDesk`) becomes
a real, connectable restricted session. From the operator's side, nothing
about how they connect changes — they use `Enter-PSSession` normally, and
JEA transparently limits what happens once they're in.

## The trap: JEA restricts *commands*, not what those commands can do

```powershell
VisibleCmdlets = 'Copy-Item'
```

Granting `Copy-Item` without constraining its parameters lets the
operator copy *any* file the virtual account can reach to *any*
destination it can write to — including files well outside whatever
narrow task you had in mind. JEA is only as tight as the parameter
constraints on each granted command; a bare cmdlet name in
`VisibleCmdlets` is often far broader than intended. Always ask "what's
the most this command could do with arbitrary arguments?" before granting
it unconstrained.

## The trap: forgetting to test as the restricted role, not as admin

A `.psrc`/`.pssc` pair that looks correct on paper can still be wrong —
the only real test is connecting *as a member of the restricted group*
and confirming `Get-Command` inside that session shows exactly the
intended, minimal list:

```powershell
Enter-PSSession -ComputerName Server01 -ConfigurationName ServiceDesk
Get-Command   # should show ONLY the granted cmdlets/functions
```

Testing as an administrator, or testing the `.psrc`/`.pssc` files by
inspection alone, misses the actual attack surface — you have to see the
restricted session from the inside.

## Cheat sheet

| Piece | Purpose |
|---|---|
| `.psrc` (role capability file) | allow-list of cmdlets/functions/programs for one role |
| `.pssc` (session configuration file) | binds role(s) to security group(s), sets session type |
| `SessionType RestrictedRemoteServer` | minimal baseline, nothing extra unless granted |
| `RunAsVirtualAccount` | connecting user's identity never runs the commands |
| `RoleDefinitions` | maps a security group to a role capability |
| Parameter-constrained `VisibleCmdlets` | narrows a broad cmdlet to specific safe arguments |
| `Register-PSSessionConfiguration` | activates the endpoint (Windows/WinRM) |
| Testing as the restricted role | the only real verification of what's actually exposed |

## Exercise

Design (as `.psrc`/`.pssc` files, without needing a live WinRM endpoint to
verify) a JEA role for a "backup operator": allowed to run a specific
backup script and check its own job status, but nothing else — including
constraining any file-related cmdlet you grant to a single specific
directory via a parameter `ValidateSet` or `ValidatePattern`. Write out
what you'd expect `Get-Command` to show inside that restricted session,
and double-check each granted command against "what's the most this could
do with arbitrary arguments" before finalizing it.
