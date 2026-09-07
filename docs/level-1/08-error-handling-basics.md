# 08 · Error Handling Basics

## Terminating vs non-terminating errors

PowerShell has two kinds of errors. **Non-terminating** errors (the default
for most cmdlets) print an error message but let the script keep running.
**Terminating** errors stop execution of the current scope immediately
unless caught.

```powershell
Get-Item "does-not-exist.txt"
Write-Output "This line still runs"
```

```text
Get-Item: Cannot find path '.../does-not-exist.txt' because it does not exist.
This line still runs
```

## try / catch / finally

```powershell
try {
    $result = 10 / 0
} catch {
    Write-Output "Something went wrong: $($_.Exception.Message)"
} finally {
    Write-Output "This always runs, error or not"
}
```

```text
Something went wrong: Attempted to divide by zero.
This always runs, error or not
```

`try/catch` only intercepts **terminating** errors by default. Many built-in
cmdlets raise non-terminating errors, which `catch` will silently miss unless
you force them to terminate with `-ErrorAction Stop`:

```powershell
try {
    Get-Item "does-not-exist.txt" -ErrorAction Stop
} catch {
    Write-Output "Caught it: $($_.Exception.Message)"
}
```

```text
Caught it: Cannot find path '...\does-not-exist.txt' because it does not exist.
```

## Inspecting the caught error

```powershell
try {
    1 / 0
} catch {
    Write-Output "Message: $($_.Exception.Message)"
    Write-Output "Type: $($_.Exception.GetType().Name)"
    Write-Output "Line: $($_.InvocationInfo.ScriptLineNumber)"
}
```

Inside a `catch` block, `$_` (or `$PSItem`) refers to the
`ErrorRecord` that was thrown — it carries the underlying `.Exception`
plus metadata about where it happened.

## Catching specific exception types

```powershell
try {
    [int]"not a number"
} catch [System.FormatException] {
    Write-Output "That wasn't a valid number"
} catch {
    Write-Output "Some other error: $($_.Exception.Message)"
}
```

More specific `catch` blocks should come first — PowerShell checks them in
order and uses the first one whose type matches.

## Throwing your own errors

```powershell
function Set-Age {
    param([int]$Age)
    if ($Age -lt 0) {
        throw "Age cannot be negative: $Age"
    }
    Write-Output "Age set to $Age"
}

try {
    Set-Age -Age -5
} catch {
    Write-Output "Rejected: $($_.Exception.Message)"
}
```

## The $Error automatic variable

Every terminating (and many non-terminating) error also gets recorded in the
`$Error` collection for the session, most recent first — handy for
post-mortem debugging in an interactive session.

```powershell
$Error.Clear()          # start fresh

Get-Item "missing.txt" -ErrorAction SilentlyContinue

$Error.Count             # 1
$Error[0].Exception.Message
```

## -ErrorAction: controlling non-terminating errors

```powershell
Get-Item "missing.txt" -ErrorAction SilentlyContinue    # suppress the error entirely
Get-Item "missing.txt" -ErrorAction Stop                 # promote to terminating (catchable)
Get-Item "missing.txt" -ErrorAction Continue              # default: show it, keep going
Get-Item "missing.txt" -ErrorAction Inquire                # ask the user what to do
```

## A robust pattern

```powershell
function Read-ConfigFile {
    param([string]$Path)

    try {
        if (-not (Test-Path $Path)) {
            throw "Config file not found: $Path"
        }
        $content = Get-Content -Path $Path -Raw -ErrorAction Stop
        return $content | ConvertFrom-Json
    } catch {
        Write-Output "Failed to load config: $($_.Exception.Message)"
        return $null
    }
}

$config = Read-ConfigFile -Path "settings.json"
if ($null -eq $config) {
    Write-Output "Using default settings instead"
}
```

## Cheat sheet

| Concept | Meaning |
|---------|---------|
| Terminating error | stops the current scope; catchable with `try/catch` |
| Non-terminating error | prints and continues; needs `-ErrorAction Stop` to be catchable |
| `try { } catch { } finally { }` | structured error handling |
| `$_` / `$PSItem` in `catch` | the caught `ErrorRecord` |
| `throw "message"` | raise your own terminating error |
| `$Error` | session-wide list of past errors, most recent first |
| `-ErrorAction Stop/SilentlyContinue/Continue/Inquire` | control how non-terminating errors behave |

## How It Actually Works

PowerShell has two genuinely different error mechanisms sharing one
vocabulary, and confusing them is the single biggest source of "my
try/catch didn't catch it" bugs. A **terminating error** unwinds the call
stack as a real CLR exception (`RuntimeException` subclasses like
`RuntimeException`, `CommandNotFoundException`, or your own `throw`'d
object wrapped in `RuntimeException`) — this is what `try/catch` is built
to intercept, because `catch` is compiled to an actual CLR `catch` block
around the `try` script block's execution. A **non-terminating error**,
by contrast, is a cmdlet calling `WriteError()` on its internal
`Cmdlet` API — it appends an `ErrorRecord` to the error stream and the
cmdlet's `ProcessRecord` *keeps running*; no exception is thrown, so a
bare `try/catch` around it does nothing at all, which is exactly why
`Get-ChildItem BadPath -ErrorAction Stop` (forcing the non-terminating
error to escalate into a terminating one) or checking `-ErrorVariable`
afterward are the two real ways to handle it.

Every error, terminating or not, materializes as an `ErrorRecord` — a
structured object holding the `Exception`, a `CategoryInfo` (a coarse
classification like `ObjectNotFound` or `PermissionDenied` used for
`-ErrorAction`/filtering), a `FullyQualifiedErrorId` (a stable string
identifier meant for programmatic matching, unlike the free-text message),
and `InvocationInfo` (line number, script path, offending command) — this
structured shape is why `catch { $_.Exception.Message }` and
`catch { $_.CategoryInfo.Reason }` are both meaningful on the same caught
object: `$_` in a catch block is the `ErrorRecord`, not the raw exception.

`-ErrorAction` is implemented as a **common parameter** injected into
every cmdlet's parameter set by the engine (not opted into per-cmdlet), and
it works by wrapping the cmdlet's calls to `WriteError` — `Stop` makes
`WriteError` throw instead of append, `SilentlyContinue` makes it a no-op,
`Continue` (the default) appends to `$Error`/the error stream and
displays it. `$Error` itself is a fixed-capacity queue
(`$Error.Count` capped by `$MaximumErrorCount`, default 256) maintained by
the engine on every error of either kind, which is why it accumulates
across an entire session, not just the current statement.

## Exercise

Write `safe-divide.ps1` containing a function `Invoke-SafeDivide` that takes
two `[double]` parameters, throws a custom error message if the divisor is
zero, and otherwise returns the result. Wrap a few test calls (including a
divide-by-zero case) in `try/catch` and print either the result or a friendly
error message for each.
