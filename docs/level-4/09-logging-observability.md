# 09 · Logging & Observability for Scripts

`Write-Host` output disappears the moment a terminal scrolls past it. A
script running unattended — a scheduled task, a CI job, a service running
on a server nobody is watching — needs its own trail: structured log
entries someone (or some log-aggregation tool) can search after the fact,
timing data that shows which step actually took the time, and a record of
exactly what failed and why. This module covers building that trail with
nothing but the PowerShell you already know.

## Structured logging instead of `Write-Host`

```powershell
function Write-Log {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string] $Message,
        [ValidateSet('INFO','WARN','ERROR')] [string] $Level = 'INFO',
        [string] $Path = './app.log'
    )
    $entry = [pscustomobject]@{
        Timestamp = (Get-Date).ToString('o')
        Level     = $Level
        Message   = $Message
    }
    $entry | ConvertTo-Json -Compress | Add-Content -Path $Path
    $entry
}

Write-Log -Message "Service starting" -Level INFO
Write-Log -Message "Cache miss for key 'user:42'" -Level WARN
Write-Log -Message "Connection refused" -Level ERROR
Get-Content ./app.log
```

```text
Timestamp                        Level Message
---------                        ----- -------
2026-08-26T11:30:17.1698980+05:30 INFO  Service starting
2026-08-26T11:30:17.2383140+05:30 WARN  Cache miss for key 'user:42'
2026-08-26T11:30:17.2398960+05:30 ERROR Connection refused

{"Timestamp":"2026-08-26T11:30:17.1698980+05:30","Level":"INFO","Message":"Service starting"}
{"Timestamp":"2026-08-26T11:30:17.2383140+05:30","Level":"WARN","Message":"Cache miss for key 'user:42'"}
{"Timestamp":"2026-08-26T11:30:17.2398960+05:30","Level":"ERROR","Message":"Connection refused"}
```

`Write-Log` does two things `Write-Host` can't: it returns the log entry
as an object (so the caller can still see it on screen, pipe it onward, or
ignore it), and it persists a **JSON line per entry** to disk. One JSON
object per line — "JSON Lines" format — is the detail that matters: a log
aggregator (Splunk, ELK, Azure Monitor, or just `Get-Content | ConvertFrom-Json`
in a pinch) can parse the file one line at a time without loading and
parsing the whole file as one JSON document, which matters once the file
has millions of lines.

## Timing operations, not just logging them

```powershell
function Invoke-Timed {
    param([Parameter(Mandatory)] [scriptblock] $Action, [string] $Name = 'operation')
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    try {
        & $Action
        $sw.Stop()
        [pscustomobject]@{ Operation = $Name; Ms = $sw.ElapsedMilliseconds; Status = 'OK' }
    } catch {
        $sw.Stop()
        [pscustomobject]@{ Operation = $Name; Ms = $sw.ElapsedMilliseconds; Status = "FAILED: $($_.Exception.Message)" }
    }
}

Invoke-Timed -Name 'sleep-fast' -Action { Start-Sleep -Milliseconds 50 }
Invoke-Timed -Name 'divide-by-zero' -Action { 1/0 }
```

```text
Operation        Ms Status
---------        -- ------
sleep-fast      241 OK
divide-by-zero   41 FAILED: Attempted to divide by zero.
```

Wrapping a step in `Invoke-Timed` gets you **both** an outcome and a
duration in one object, whether the step succeeds or throws — the
`try`/`catch` here means a failing step still returns a normal object
(with the error captured in `Status`) instead of blowing up the whole
script, so a pipeline of ten timed steps can run to completion and report
which ones failed rather than dying on the first exception. Piping the
returned objects into `Write-Log` (or straight to `Export-Csv`) turns this
into a real per-run performance record you can compare across days.

## `Start-Transcript`: capturing everything, unfiltered

```powershell
Start-Transcript -Path './session.log' -Force
Write-Output "Doing work..."
Get-Date
Stop-Transcript
```

```text
Transcript started, output file is ./session.log

Doing work...

Wednesday, August 26, 2026 11:32:04 AM

Transcript stopped, output file is ./session.log
```

`Start-Transcript` captures *everything* written to the console —
commands, their output, warnings, errors — verbatim, exactly as a user
would have seen it interactively. It's the blunt instrument compared to
`Write-Log`'s structured entries: no filtering, no levels, nothing
machine-parseable, but zero code changes needed to add it to an existing
script. Reach for it when debugging an interactive session after the
fact; reach for structured logging when a machine (or a dashboard) needs
to consume the output.

## `$ErrorActionPreference` and catching what would otherwise vanish

```powershell
$ErrorActionPreference = 'Stop'
try {
    Get-Item './does-not-exist.txt' -ErrorAction Stop
} catch {
    Write-Log -Message $_.Exception.Message -Level ERROR
    Write-Log -Message "at $($_.InvocationInfo.ScriptName):$($_.InvocationInfo.ScriptLineNumber)" -Level ERROR
}
```

```text
{"Timestamp":"2026-08-26T11:33:02.1120450+05:30","Level":"ERROR","Message":"Cannot find path '/does-not-exist.txt' because it does not exist."}
{"Timestamp":"2026-08-26T11:33:02.1134210+05:30","Level":"ERROR","Message":"at :3"}
```

A non-terminating error (the default for most cmdlets) can scroll past in
a long-running unattended script without anyone noticing — the script
keeps going, but silently skipped a step. Setting `$ErrorActionPreference
= 'Stop'` (or `-ErrorAction Stop` per-command) turns those into
terminating errors a `catch` block can actually see and log, with
`$_.InvocationInfo` giving the exact line where it happened — essential
once the script runs somewhere you can't watch it live.

## The trap: logging inside a tight loop kills performance

```powershell
# Slow: one Add-Content (one file open/write/close) per iteration
1..1000 | ForEach-Object { Write-Log -Message "processing item $_" -Path './loop.log' }
```

`Add-Content` opens, writes, and closes the file on every single call.
Logging one line per iteration of a loop processing thousands of items
turns a fast in-memory operation into thousands of file-system round
trips — often the actual bottleneck, not whatever the loop is supposedly
doing. Batch it instead: accumulate entries in an array and write once.

```powershell
$entries = 1..1000 | ForEach-Object {
    [pscustomobject]@{ Timestamp = (Get-Date).ToString('o'); Level = 'INFO'; Message = "processing item $_" }
}
$entries | ConvertTo-Json -Compress | Set-Content -Path './loop.log'
```

## Cheat sheet

| Need | Tool |
|---|---|
| Structured, machine-parseable entries | custom `Write-Log` writing JSON Lines |
| Time an operation, capture success/failure | `[System.Diagnostics.Stopwatch]` + `try`/`catch` |
| Capture an entire interactive session verbatim | `Start-Transcript` / `Stop-Transcript` |
| Make non-terminating errors catchable | `$ErrorActionPreference = 'Stop'` or `-ErrorAction Stop` |
| Find where an error happened | `$_.InvocationInfo.ScriptLineNumber` |
| Avoid the tight-loop logging trap | batch entries, write once instead of per-iteration |

## Exercise

Add a `Write-Log` function to the `AdminToolkit` module from Level 3's
project. Wrap each of its existing public functions' bodies in
`Invoke-Timed`, logging the resulting `Operation`/`Ms`/`Status` object at
`INFO` level on success and `ERROR` level on failure (with the caught
exception's message). Run the module against at least one input that
succeeds and one that throws, then inspect the resulting log file and
confirm both a timed success entry and a timed failure entry are present
as valid JSON Lines.
