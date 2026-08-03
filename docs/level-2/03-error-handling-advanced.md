# 03 · Error Handling Advanced

[Level 1](../level-1/08-error-handling-basics.md) covered `try`/`catch`/
`finally` and `-ErrorAction`. This module goes further into how PowerShell's
dual error system actually works end to end, how to build and throw your
own structured exceptions, and the mistakes that make error handling look
correct while silently doing nothing.

## Terminating vs. non-terminating, precisely

Every error in PowerShell carries a `CategoryInfo` and is either
**terminating** (stops the current pipeline/scope unless caught) or
**non-terminating** (reported, then execution continues). The type of
error a cmdlet raises is a design decision made by whoever wrote it — not
something you can tell just by reading the call.

```powershell
# Get-ChildItem raises a NON-terminating error for a missing path by default
Get-ChildItem "Z:\does-not-exist" 2>$null
Write-Output "Still running"
```

```text
Still running
```

```powershell
# Force it to terminate so try/catch can see it
try {
    Get-ChildItem "Z:\does-not-exist" -ErrorAction Stop
} catch {
    Write-Output "Caught: $($_.Exception.Message)"
}
```

```text
Caught: Cannot find path 'Z:\does-not-exist' because it does not exist.
```

The trap here is assuming `try/catch` around a cmdlet automatically works —
if that cmdlet's errors are non-terminating and you didn't add
`-ErrorAction Stop`, the `catch` block simply never runs, and the error
just prints and gets swallowed by whatever collected the output.

## `$ErrorActionPreference`: the script-wide default

Instead of adding `-ErrorAction Stop` to every single cmdlet call, you can
set the default for the whole script or scope.

```powershell
$ErrorActionPreference = "Stop"    # every cmdlet now terminates on error by default

try {
    Get-ChildItem "Z:\does-not-exist"
    Get-Content "also-missing.txt"
} catch {
    Write-Output "Caught: $($_.Exception.Message)"
}
```

```text
Caught: Cannot find path 'Z:\does-not-exist' because it does not exist.
```

Only the *first* failing line ran — once `$ErrorActionPreference = "Stop"`
promotes errors to terminating, the `catch` block catches the first one and
skips the rest of the `try` block, same as it would with any terminating
error.

| Value | Behavior |
|---|---|
| `"Continue"` (default) | show non-terminating errors, keep going |
| `"Stop"` | promote all non-terminating errors to terminating |
| `"SilentlyContinue"` | suppress non-terminating errors entirely |
| `"Inquire"` | prompt interactively on every error |

Setting `$ErrorActionPreference = "Stop"` at the top of a script is a common
and reasonable pattern for automation scripts where "fail fast and loud" is
the desired behavior — much safer than a script that silently limps along
after something has already gone wrong.

## Restoring the previous preference

```powershell
function Invoke-Strict {
    param([scriptblock]$Action)

    $previous = $ErrorActionPreference
    $ErrorActionPreference = "Stop"
    try {
        & $Action
    } finally {
        $ErrorActionPreference = $previous
    }
}
```

Changing `$ErrorActionPreference` is scoped to wherever you set it (a
function's local scope, if set inside one) — but if you set it at script
scope and other code relies on the old value, save and restore it
explicitly, as above, rather than assuming it "resets itself."

## Catching by exception type — and why order matters

```powershell
function Invoke-Parse {
    param([string]$Value)

    try {
        [int]::Parse($Value)
    }
    catch [System.FormatException] {
        Write-Output "Not a valid number: '$Value'"
    }
    catch [System.OverflowException] {
        Write-Output "Number too large: '$Value'"
    }
    catch {
        Write-Output "Unexpected error: $($_.Exception.GetType().Name)"
    }
}

Invoke-Parse "abc"           # Not a valid number: 'abc'
Invoke-Parse "99999999999999999999"   # Number too large: '99999999999999999999'
```

`catch` blocks are checked top-to-bottom and the **first matching type**
wins — including base classes. If a generic `catch [System.Exception]`
block were listed first, it would swallow every error type below it, so
always order from most specific to least specific, with the untyped
`catch { }` last as a safety net.

## Building your own exception types

For scripts and modules that need callers to distinguish *your* errors from
generic .NET ones, throw a real exception object instead of a bare string.

```powershell
class InsufficientFundsException : System.Exception {
    [decimal]$Requested
    [decimal]$Available

    InsufficientFundsException([decimal]$requested, [decimal]$available) :
        base("Requested $requested but only $available is available") {
        $this.Requested = $requested
        $this.Available = $available
    }
}

function Invoke-Withdrawal {
    param([decimal]$Amount, [decimal]$Balance)

    if ($Amount -gt $Balance) {
        throw [InsufficientFundsException]::new($Amount, $Balance)
    }
    return $Balance - $Amount
}

try {
    Invoke-Withdrawal -Amount 500 -Balance 200
} catch [InsufficientFundsException] {
    Write-Output "Blocked: $($_.Exception.Message)"
    Write-Output "Short by $($_.Exception.Requested - $_.Exception.Available)"
}
```

```text
Blocked: Requested 500 but only 200 is available
Short by 300
```

A custom exception class lets calling code `catch` specifically for your
error type — and carry structured data (`$Requested`, `$Available`) with
it, instead of forcing everyone to regex-parse a message string.

## `throw` vs `Write-Error`: two different tools

```powershell
function Set-Score {
    param([int]$Score)

    if ($Score -lt 0) {
        throw "Score cannot be negative"     # terminating: stops the function immediately
    }
    if ($Score -gt 100) {
        Write-Error "Score $Score exceeds 100, clamping"   # non-terminating by default
        $Score = 100
    }
    return $Score
}
```

`throw` always raises a terminating error — the function stops right there
unless something catches it. `Write-Error` raises a non-terminating error
by default: it reports a problem but lets execution continue on the next
line, which is appropriate for "this is wrong, but I can recover and keep
going" situations.

## Re-throwing without losing the original error

```powershell
try {
    try {
        1 / 0
    } catch {
        Write-Output "Logging: $($_.Exception.Message)"
        throw    # re-throws the SAME error, preserving the original stack trace
    }
} catch {
    Write-Output "Outer caught: $($_.Exception.Message)"
}
```

A bare `throw` (no argument) inside a `catch` block re-raises the exact
error that was caught — useful for "log it here, but let a caller further
up decide what to do about it." Using `throw $_.Exception.Message` instead
would create a brand-new, generic exception and lose the original type and
stack trace.

## `trap`: the older, scope-wide alternative

```powershell
function Get-RiskyValue {
    trap {
        Write-Output "Trapped: $($_.Exception.Message)"
        continue    # continue execution after the statement that failed
    }
    1 / 0
    Write-Output "This still runs because of 'continue' above"
}

Get-RiskyValue
```

`trap` predates `try/catch` and handles every terminating error in its
scope without wrapping specific statements — you'll mostly encounter it
reading older scripts. Prefer `try/catch/finally` for new code; it's more
readable about exactly which statements are being guarded.

## Cheat sheet

| Concept | Behavior |
|---|---|
| Terminating error | stops the current scope; catchable with `try/catch` |
| Non-terminating error | reports and continues; needs `-ErrorAction Stop` to be catchable |
| `$ErrorActionPreference = "Stop"` | promotes all cmdlets' errors to terminating by default |
| `catch [SpecificType] { }` | order most-specific to least-specific, generic `catch` last |
| Custom exception class | lets callers `catch` your error type and read structured data |
| `throw "msg"` | raises a new terminating error |
| `throw` (bare, in `catch`) | re-raises the original error, preserving its type/stack |
| `Write-Error` | non-terminating; reports and continues |
| `trap { }` | older scope-wide error handler; prefer `try/catch` in new code |

## Exercise

Create a class `ValidationException` (inheriting `System.Exception`) with an
extra `[string]$FieldName` property. Write a function `Test-AgeInput` that
throws a `ValidationException` (with `FieldName = "Age"`) if a given `-Age`
parameter is negative or over 130. Call it inside a `try/catch
[ValidationException]` block that prints `"Invalid <FieldName>: <message>"`,
and test it with both a valid and an invalid age.
