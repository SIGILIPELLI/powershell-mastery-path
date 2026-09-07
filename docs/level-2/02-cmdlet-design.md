# 02 · Cmdlet Design

Level 1 showed `[CmdletBinding()]` briefly for pipeline input. This module
covers what it means to design a function as a proper, well-behaved
cmdlet — the naming rules, parameter conventions, and built-in behaviors
that make a script feel like it belongs alongside `Get-Process` and
`Sort-Object` instead of a one-off script someone hacked together.

## Approved verbs aren't just a style preference

PowerShell enforces a **Verb-Noun** naming convention, and the verb must
come from a fixed, documented list (`Get`, `Set`, `New`, `Remove`, `Test`,
`Invoke`, `ConvertTo`, and so on). Using an unapproved verb doesn't break
your function, but it does generate a warning and hurts discoverability —
tab-completion, `Get-Command`, and other scripts all assume the convention.

```powershell
Get-Verb | Select-Object -First 5 -Property Verb, Group
```

```text
Verb    Group
----    -----
Add     Common
Clear   Common
Close   Common
Copy    Common
Enter   Common
```

```powershell
function Create-Report {   # "Create" is not an approved verb
    Write-Output "..."
}
```

```text
WARNING: The names of some imported commands from the module 'x' include
unapproved verbs that might make them less discoverable.
```

The fix is almost always to pick the closest approved verb — `New-Report`,
not `Create-Report`; `Get-Report`, not `Fetch-Report` or `Retrieve-Report`.

| Instead of... | Use |
|---|---|
| `Create`, `Make` | `New` |
| `Delete`, `Destroy`, `Kill` | `Remove` |
| `Change`, `Modify`, `Update` | `Set` |
| `Fetch`, `Retrieve`, `Read` | `Get` |
| `Run`, `Execute`, `Start` | `Invoke` (one-off action) or `Start` (long-running) |
| `Check`, `Validate` | `Test` (must return a boolean) |

## `[CmdletBinding()]` unlocks common parameters

```powershell
function Write-Status {
    [CmdletBinding()]
    param(
        [string]$Message
    )
    Write-Verbose "About to write status"
    Write-Output $Message
}

Write-Status -Message "Deploy complete" -Verbose
```

```text
VERBOSE: About to write status
Deploy complete
```

Without `[CmdletBinding()]`, `-Verbose`, `-Debug`, `-ErrorAction`,
`-WarningAction`, and `-OutVariable` don't exist on your function at all.
With it, they're free — and `Write-Verbose`/`Write-Warning` calls inside the
function automatically respect them.

## Parameter validation attributes

Validating input inside the function body with `if` statements works, but
attributes catch bad input **before** the function body even runs, with a
consistent, informative error message.

```powershell
function New-Employee {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [string]$Name,

        [ValidateRange(18, 70)]
        [int]$Age,

        [ValidateSet("Engineering", "Sales", "Support")]
        [string]$Department,

        [ValidatePattern("^[A-Z]{2}\d{4}$")]
        [string]$EmployeeId,

        [ValidateScript({ Test-Path $_ })]
        [string]$PhotoPath
    )

    [pscustomobject]@{
        Name       = $Name
        Age        = $Age
        Department = $Department
        EmployeeId = $EmployeeId
    }
}

New-Employee -Name "Priya" -Age 29 -Department "Engineering" -EmployeeId "EN2045"
```

```powershell
New-Employee -Name "Priya" -Age 29 -Department "Marketing" -EmployeeId "EN2045"
```

```text
New-Employee: Cannot validate argument on parameter 'Department'. The
argument "Marketing" does not belong to the set "Engineering,Sales,Support"
specified by the ValidateSet attribute.
```

| Attribute | Checks |
|---|---|
| `[ValidateNotNullOrEmpty()]` | value isn't `$null` or `""` |
| `[ValidateRange(min,max)]` | numeric value falls in range |
| `[ValidateSet("a","b")]` | value is one of a fixed list |
| `[ValidatePattern("regex")]` | value matches a regular expression |
| `[ValidateScript({ ... })]` | value passes a custom script block (must return `$true`) |
| `[ValidateLength(min,max)]` | string length falls in range |

## Parameter sets: mutually exclusive options

```powershell
function Get-Report {
    [CmdletBinding(DefaultParameterSetName = "ByDate")]
    param(
        [Parameter(Mandatory, ParameterSetName = "ByDate")]
        [datetime]$Date,

        [Parameter(Mandatory, ParameterSetName = "ByRange")]
        [datetime]$StartDate,

        [Parameter(Mandatory, ParameterSetName = "ByRange")]
        [datetime]$EndDate
    )

    if ($PSCmdlet.ParameterSetName -eq "ByDate") {
        Write-Output "Report for $($Date.ToShortDateString())"
    } else {
        Write-Output "Report from $($StartDate.ToShortDateString()) to $($EndDate.ToShortDateString())"
    }
}

Get-Report -Date (Get-Date)
Get-Report -StartDate "2026-01-01" -EndDate "2026-01-31"
```

Parameter sets let one cmdlet expose two (or more) different, mutually
exclusive ways of calling it — PowerShell rejects a call that mixes
parameters from different sets, and `$PSCmdlet.ParameterSetName` tells you
which one was actually used, inside the function.

## Should-process: supporting `-WhatIf` and `-Confirm`

Any cmdlet that changes system state (deletes files, stops processes,
modifies data) should support `-WhatIf`/`-Confirm` the way built-in cmdlets
like `Remove-Item` do.

```powershell
function Remove-TempFile {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory)]
        [string]$Path
    )

    if ($PSCmdlet.ShouldProcess($Path, "Remove file")) {
        Remove-Item -Path $Path
        Write-Output "Removed $Path"
    }
}

Remove-TempFile -Path "cache.tmp" -WhatIf
```

```text
What if: Performing the operation "Remove file" on target "cache.tmp".
```

`SupportsShouldProcess = $true` plus wrapping the actual side-effecting code
in `if ($PSCmdlet.ShouldProcess(...))` is the entire pattern — PowerShell
handles the `-WhatIf`/`-Confirm` prompting and messaging for you.

## The trap: forgetting `OutputType` leaves callers guessing

```powershell
function Get-ActiveUser {
    [CmdletBinding()]
    [OutputType([pscustomobject])]
    param(
        [string]$Department
    )
    [pscustomobject]@{ Name = "Ada"; Department = $Department }
}
```

`[OutputType(...)]` doesn't enforce anything at runtime, but it documents
the return type for `Get-Command -Syntax`, IntelliSense, and other tooling
that inspects your function without running it — cheap to add, and it pays
off the moment someone else (or future you) has to use the function without
rereading its body.

## Naming and casing conventions

| Rule | Example |
|---|---|
| `Verb-Noun`, singular noun | `Get-User`, not `Get-Users` |
| PascalCase for public names | `Get-EmployeeReport` |
| Parameters are PascalCase, no abbreviations | `-EmployeeId`, not `-empid` |
| Switch parameters read as a yes/no flag | `-Force`, `-WhatIf`, not `-ForceFlag` |
| Boolean-returning functions use `Test-` | `Test-Path`, `Test-Connection` |

## Cheat sheet

| Concept | Purpose |
|---|---|
| `Get-Verb` | list PowerShell's approved verbs |
| `[CmdletBinding()]` | unlocks `-Verbose`/`-Debug`/`-ErrorAction`/etc. |
| `[ValidateNotNullOrEmpty()]`, `[ValidateRange]`, `[ValidateSet]`, `[ValidatePattern]`, `[ValidateScript]` | fail fast on bad input |
| `ParameterSetName` | mutually exclusive groups of parameters |
| `$PSCmdlet.ParameterSetName` | which set was actually used |
| `SupportsShouldProcess` + `$PSCmdlet.ShouldProcess(...)` | enables `-WhatIf`/`-Confirm` |
| `[OutputType(...)]` | documents the return type for tooling |

## How It Actually Works

Parameter sets are resolved through a real constraint-satisfaction pass,
not a simple "which parameters did you type" match. When you invoke a
function with multiple `[Parameter(ParameterSetName=...)]` groups, the
binder builds the set of candidate parameter sets that are consistent
with every named argument you supplied, then eliminates candidates whose
mandatory parameters weren't satisfied. If more than one candidate
survives, PowerShell tries to disambiguate using `DefaultParameterSetName`
(declared on `[CmdletBinding()]`); if that still doesn't resolve to
exactly one, you get `ParameterBindingException: Parameter set cannot be
resolved` — the same error a compiled cmdlet throws, because both paths
run through `System.Management.Automation.CommandProcessorBase`'s shared
parameter-binding pipeline.

`ValidateSet`, `ValidateRange`, `ValidateScript`, and friends are not
convention-based checks the function body has to remember to call —
they're `ValidateArgumentsAttribute` subclasses whose `Validate()` method
the binder invokes automatically, immediately after type coercion and
before your function body runs at all. This ordering is why a
`ValidateScript` block can throw a validation error before a single line
of your function executes, and why validation attributes stack: multiple
`[Validate*]` attributes on one parameter all run, in declaration order,
and the first failure wins.

`SupportsShouldProcess` wires `-WhatIf`/`-Confirm` into the same
`$PSCmdlet.ShouldProcess()` call compiled cmdlets use — calling it
consults the current `ConfirmPreference` and any `-WhatIf`/`-Confirm`
switch bound for this invocation, and if `-WhatIf` is in effect, it
**returns `$false` without running your side-effecting code at all**;
this is why the well-known pattern `if ($PSCmdlet.ShouldProcess(...)) {
<risky work> }` is not just good style — skipping the `if` guard means
`-WhatIf` has no effect, since the destructive work isn't actually gated
on anything.

Cmdlet/function naming (`Verb-Noun`) is enforced at the module-analysis
level: `Get-Verb` reflects the engine's approved-verb table, and importing
a module with unapproved verbs triggers a non-fatal warning (not an
error) because the check happens in `Import-Module`'s post-load
validation pass over exported command names, purely for discoverability
— it has no effect on whether the function actually runs.

## Exercise

Write a function `New-Ticket` with `[CmdletBinding(SupportsShouldProcess =
$true)]`. It should take a mandatory `-Title` (`[ValidateNotNullOrEmpty()]`),
a `-Priority` restricted to `"Low"`, `"Medium"`, `"High"` via
`[ValidateSet(...)]`, and default to `"Medium"`. Inside, use
`$PSCmdlet.ShouldProcess($Title, "Create ticket")` before "creating" the
ticket (just `Write-Output` a confirmation). Test it both normally and with
`-WhatIf`.
