# 01 · Advanced Functions & Parameter Validation

Level 1 and 2 functions took whatever was handed to them and let the body
sort it out — a bad value would surface as a confusing error three lines
deep, or worse, silently produce garbage. Advanced functions push
validation up to the parameter itself: bad input is rejected before your
code ever runs, with a message that names the exact problem.

## `[CmdletBinding()]`: the switch that turns a function "advanced"

```powershell
function Test-EvenNumber {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [int]$Number
    )
    process {
        $Number % 2 -eq 0
    }
}
```

```powershell
1..6 | Test-EvenNumber
```

```text
False
True
False
True
False
True
```

`[CmdletBinding()]` gives a plain function the same behaviors as a real
cmdlet: common parameters (`-Verbose`, `-ErrorAction`, `-WhatIf` with
`SupportsShouldProcess`), strict parameter binding, and support for
`begin`/`process`/`end` blocks. Without it, `param()` still works, but
pipeline input and `-Verbose` do not.

## Validation attributes

Each one rejects bad input at the parameter, before the function body
runs at all:

```powershell
function New-UserAccount {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        [ValidatePattern('^[a-zA-Z][a-zA-Z0-9._-]{2,19}$')]
        [string]$UserName,

        [Parameter(Mandatory)]
        [ValidateSet('Standard', 'Admin', 'Guest')]
        [string]$Role,

        [ValidateRange(13, 120)]
        [int]$Age = 18,

        [ValidateScript({
            if ($_ -match '^\S+@\S+\.\S+$') { $true }
            else { throw "'$_' is not a valid email address" }
        })]
        [string]$Email,

        [ValidateNotNullOrEmpty()]
        [string]$Department = "General"
    )

    process {
        [pscustomobject]@{
            UserName   = $UserName
            Role       = $Role
            Age        = $Age
            Email      = $Email
            Department = $Department
        }
    }
}
```

```powershell
New-UserAccount -UserName "jdoe" -Role Admin -Age 34 -Email "jdoe@example.com"
```

```text
UserName   : jdoe
Role       : Admin
Age        : 34
Email      : jdoe@example.com
Department : General
```

Now feed it bad input:

```powershell
try { New-UserAccount -UserName "jd" -Role Admin }
catch { "Caught: $($_.Exception.Message)" }
```

```text
Caught: Cannot validate argument on parameter 'UserName'. The argument
"jd" does not match the "^[a-zA-Z][a-zA-Z0-9._-]{2,19}$" pattern. Supply
an argument that matches "^[a-zA-Z][a-zA-Z0-9._-]{2,19}$" and try the
command again.
```

```powershell
try { New-UserAccount -UserName "jdoe2" -Role SuperAdmin }
catch { "Caught: $($_.Exception.Message)" }
```

```text
Caught: Cannot validate argument on parameter 'Role'. The argument
"SuperAdmin" does not belong to the set "Standard,Admin,Guest" specified
by the ValidateSet attribute.
```

```powershell
try { New-UserAccount -UserName "jdoe3" -Role Standard -Email "not-an-email" }
catch { "Caught: $($_.Exception.Message)" }
```

```text
Caught: 'not-an-email' is not a valid email address
```

That last message is the custom one from inside `ValidateScript` — when
the script block throws, its message becomes the validation failure
message instead of PowerShell's generic wording. `ValidateScript` runs
once per value with `$_` bound to it, and must return `$true`/`$false`
or throw; returning `$false` produces PowerShell's generic (less useful)
error, so throwing your own message is almost always better.

| Attribute | Rejects |
|---|---|
| `ValidateNotNullOrEmpty` | `$null`, empty string, empty collection |
| `ValidateSet(...)` | anything not in the fixed list |
| `ValidateRange(min, max)` | numbers outside the range |
| `ValidatePattern('regex')` | strings that don't match the regex |
| `ValidateScript({...})` | anything the script block rejects or throws on |
| `ValidateCount(min, max)` | arrays with too few/many elements |

## `begin` / `process` / `end`: the trap of pipeline scoping

A function that accepts pipeline input runs its `process` block once
**per item**, but `begin` and `end` each run exactly once — a distinction
that trips people up constantly:

```powershell
function Get-RunningTotal {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [int]$Value
    )
    begin {
        $total = 0
        Write-Verbose "Starting total at 0"
    }
    process {
        $total += $Value
        [pscustomobject]@{ Value = $Value; RunningTotal = $total }
    }
    end {
        Write-Output "Final total: $total"
    }
}

1..4 | Get-RunningTotal
```

```text
Value RunningTotal
----- ------------
    1            1
    2            3
    3            6
    4           10
Final total: 10
```

The trap: if you write `$total = 0` inside `process` instead of `begin`,
it resets to zero on *every* piped item, and your "running" total never
accumulates. `begin` is for one-time setup (opening a connection,
initializing an accumulator); `process` is for per-item work; `end` is
for one-time cleanup or a final summary — put logic in the wrong block
and it silently runs the wrong number of times.

If a function has no `begin`/`process`/`end` blocks at all, its entire
body behaves as an implicit `end` block — meaning it only runs once,
*after* all pipeline input has already arrived, not once per item. That's
why a function using `$Number` from `ValueFromPipeline` but written as flat
top-level code will only ever see the *last* item piped to it.

## `$_` and `$PSItem` inside a function

Inside `ValidateScript` and inside pipeline-aware scriptblocks generally,
`$_` (and its more readable alias `$PSItem`) refers to the current
pipeline object being evaluated. They're interchangeable — `$PSItem` was
added later purely for readability in longer scripts:

```powershell
[ValidateScript({ $PSItem -gt 0 })]
[int]$Quantity
```

The trap: `$_`/`$PSItem` are only bound inside blocks PowerShell is
actively iterating (`ForEach-Object`, `Where-Object`, `process` via
pipeline, validation scriptblocks). Reference them outside such a
context — say, in a plain function body — and you get `$null`, not an
error, which can silently produce wrong results instead of failing loudly.

## Dynamic help with comment-based help

```powershell
function New-UserAccount {
    <#
    .SYNOPSIS
        Creates a validated user account object.
    .PARAMETER UserName
        3-20 characters, must start with a letter.
    .EXAMPLE
        New-UserAccount -UserName jdoe -Role Admin
    #>
    [CmdletBinding()]
    param(...)
}
```

`Get-Help New-UserAccount -Full` picks this up automatically — no extra
registration needed. It costs a few lines and turns your function into
something a teammate (or future you) can discover without reading the
source.

## Cheat sheet

| Feature | Purpose |
|---|---|
| `[CmdletBinding()]` | opt into common parameters, strict binding, pipeline blocks |
| `[Parameter(Mandatory)]` | require a value; prompts interactively if missing |
| `ValueFromPipeline` | bind the whole pipeline object to this parameter |
| `ValueFromPipelineByPropertyName` | bind a same-named property of the pipeline object |
| `ValidateSet`, `ValidateRange`, `ValidatePattern`, `ValidateScript` | reject bad values before the body runs |
| `begin` | runs once, before any pipeline item |
| `process` | runs once per pipeline item |
| `end` | runs once, after all pipeline items |
| `$_` / `$PSItem` | current item inside an active pipeline context only |
| Comment-based help (`<# .SYNOPSIS ... #>`) | powers `Get-Help` for your own functions |

## Exercise

Write an advanced function `New-InventoryItem` with `[CmdletBinding()]`,
pipeline support (`ValueFromPipelineByPropertyName`), and validation on at
least three parameters: `Sku` (pattern like `SKU-####`), `Quantity`
(range 0–10000), and `Category` (a fixed `ValidateSet` of your choosing).
Add a `begin` block that initializes a running count of items processed
and an `end` block that reports the total — then pipe an array of
hashtables/objects through it and confirm both the per-item validation
and the final count behave correctly.
