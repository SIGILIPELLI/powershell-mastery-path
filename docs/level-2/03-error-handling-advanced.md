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

## How It Actually Works

`trap` predates `try/catch` in PowerShell's design and works on an
entirely different mechanism: it's not a block scoped to a specific
statement, it's a **handler registered against the enclosing scope**
(script, function, or block) that the engine consults whenever a
terminating error propagates through that scope, regardless of which
statement in the scope raised it. This is why `trap` can feel "spooky
action at a distance" — it fires for an error thrown on line 40 even
though the `trap` statement is declared on line 3, because it's not
control flow, it's an error-dispatch registration on the scope object
itself.

`$_`/`$PSItem` inside a `catch` block is bound to the actual
`ErrorRecord` that terminated the `try`, retrieved from the engine's
current-error context at the moment the `catch` handler is entered — this
is distinct from `$Error[0]`, which is whatever the *last* error appended
to the session-wide `$Error` queue was; they usually agree but can
diverge if other code (a nested `-ErrorAction SilentlyContinue` call, for
instance) appended to `$Error` between the throw and your catch.

Typed `catch` blocks (`catch [System.IO.FileNotFoundException] {...}`)
work through .NET's normal exception-type matching — the runtime walks
your listed `catch` clauses top to bottom and uses `is`-style
assignability checks (a caught `IOException` matches a `catch
[System.Exception]` clause too, since it's a subtype), so **order
matters**: a broad `catch [System.Exception]` placed before a specific
`catch [System.IO.FileNotFoundException]` will swallow everything and the
specific clause becomes dead code, exactly as in C#/Java exception
ordering.

`finally` is guaranteed to run via the CLR's own structured exception
handling — it executes whether the `try` completed normally, threw, or
even if a `catch` block that ran the value threw again while handling the
first exception, because it's compiled to sit in the `finally` clause of
the actual IL-level `try/catch/finally` the script block's execution is
wrapped in, not reimplemented as script-level bookkeeping.

## Exercise

Create a class `ValidationException` (inheriting `System.Exception`) with an
extra `[string]$FieldName` property. Write a function `Test-AgeInput` that
throws a `ValidationException` (with `FieldName = "Age"`) if a given `-Age`
parameter is negative or over 130. Call it inside a `try/catch
[ValidationException]` block that prints `"Invalid <FieldName>: <message>"`,
and test it with both a valid and an invalid age.
