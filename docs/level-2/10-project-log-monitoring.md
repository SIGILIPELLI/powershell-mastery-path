# 10 · Project — Log Monitoring Script

Time to combine everything from Level 2 into one working tool: a **log
monitoring script** that reads `.log` files from a directory, parses each
line with a [regex](04-regular-expressions.md), builds a summary using
[advanced pipeline functions](01-advanced-pipeline.md), handles malformed
input and missing files with proper
[error handling](03-error-handling-advanced.md), ships as a real
[module with a manifest](08-script-modules-manifests.md), and has
[Pester tests](06-testing-pester.md) covering its core logic.

## Goal

Build a `LogMonitor` module plus a `Watch-Logs.ps1` entry-point script
that:

1. Parses log lines matching `"<date> <time> <LEVEL> <message>"` (e.g.
   `2026-08-02 09:45:10 ERROR Failed to connect to db`) into structured
   objects, using named regex capture groups.
2. Skips malformed lines with a warning instead of crashing the whole run.
3. Summarizes total entries, counts per level, and an overall error rate.
4. Flags `ERROR` entries that happened recently (within a configurable
   window of the newest log entry).
5. Writes both a console report and a JSON file.
6. Has a Pester test suite covering the parsing and summary logic.

## Project layout

```text
LogMonitorProject/
  LogMonitor/
    LogMonitor.psm1      # the module: parsing, summary, recent-errors logic
    LogMonitor.psd1       # manifest (generated in the same way as Module 08)
  Tests/
    LogMonitor.Tests.ps1   # Pester tests for the module's functions
  Watch-Logs.ps1            # entry-point script
  logs/
    app.log
    worker.log
```

## Step 1 — the module: `LogMonitor/LogMonitor.psm1`

```powershell
# LogMonitor/LogMonitor.psm1
$script:LogLinePattern = '^(?<Date>\d{4}-\d{2}-\d{2}) (?<Time>\d{2}:\d{2}:\d{2}) (?<Level>INFO|WARN|ERROR) (?<Message>.+)$'

function ConvertFrom-LogLine {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline = $true)]
        [string]$Line
    )

    process {
        if ($Line -match $script:LogLinePattern) {
            [pscustomobject]@{
                Timestamp = [datetime]::Parse("$($Matches.Date) $($Matches.Time)")
                Level     = $Matches.Level
                Message   = $Matches.Message
                RawLine   = $Line
            }
        } else {
            Write-Warning "Skipping unparseable line: $Line"
        }
    }
}

function Get-LogEntries {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline = $true, ValueFromPipelineByPropertyName = $true)]
        [Alias("FullName")]
        [string]$Path
    )

    process {
        try {
            if (-not (Test-Path -Path $Path -PathType Leaf)) {
                throw "Log file not found: $Path"
            }

            Get-Content -Path $Path -ErrorAction Stop | ConvertFrom-LogLine
        } catch {
            Write-Error "Failed to read '$Path': $($_.Exception.Message)"
        }
    }
}

function Get-LogSummary {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline = $true)]
        [psobject[]]$Entry
    )

    begin {
        $all = [System.Collections.Generic.List[psobject]]::new()
    }
    process {
        foreach ($e in $Entry) { $all.Add($e) }
    }
    end {
        $total = $all.Count
        $byLevel = $all | Group-Object -Property Level | Sort-Object Count -Descending

        $errorCount = ($byLevel | Where-Object Name -eq "ERROR" | Select-Object -ExpandProperty Count)
        if (-not $errorCount) { $errorCount = 0 }

        [pscustomobject]@{
            TotalEntries = $total
            ByLevel      = $byLevel | Select-Object Name, Count
            ErrorCount   = $errorCount
            ErrorRate    = if ($total -gt 0) { [math]::Round($errorCount / $total * 100, 1) } else { 0 }
        }
    }
}

function Get-RecentErrors {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline = $true)]
        [psobject[]]$Entry,

        [int]$WithinMinutes = 60
    )

    begin {
        $all = [System.Collections.Generic.List[psobject]]::new()
    }
    process {
        foreach ($e in $Entry) { $all.Add($e) }
    }
    end {
        if ($all.Count -eq 0) { return }

        $latest = ($all | Measure-Object -Property Timestamp -Maximum).Maximum
        $cutoff = $latest.AddMinutes(-$WithinMinutes)

        $all | Where-Object { $_.Level -eq "ERROR" -and $_.Timestamp -ge $cutoff } |
            Sort-Object Timestamp
    }
}

Export-ModuleMember -Function ConvertFrom-LogLine, Get-LogEntries, Get-LogSummary, Get-RecentErrors
```

A few design points worth calling out:

- `ConvertFrom-LogLine` is a `begin`/`process` pipeline function (Module
  01) built around a single named-capture regex (Module 04) — one pattern,
  reused for every line.
- `Get-LogEntries` wraps the risky part (a missing/unreadable file) in
  `try/catch` with `-ErrorAction Stop` on `Get-Content` (Module 03), so a
  bad path reports a clear error instead of an unhandled exception killing
  the whole scan.
- Malformed *individual lines* are handled differently from missing
  *files*: a bad line is a `Write-Warning` and gets skipped (recoverable,
  expected to happen occasionally in real logs); a missing file is a
  `throw`/`Write-Error` (a real problem worth surfacing loudly).
- `Get-LogSummary` and `Get-RecentErrors` both buffer their input in `end`
  rather than computing incrementally in `process`, because both need the
  *complete* set of entries before they can answer questions like "what's
  the newest timestamp" or "what fraction were errors."

## Step 2 — the manifest: `LogMonitor/LogMonitor.psd1`

Generated the same way as [Module 08](08-script-modules-manifests.md):

```powershell
New-ModuleManifest -Path "./LogMonitor/LogMonitor.psd1" `
    -RootModule "LogMonitor.psm1" `
    -ModuleVersion "1.0.0" `
    -Author "Your Name" `
    -Description "Parses and summarizes application log files" `
    -FunctionsToExport @("ConvertFrom-LogLine", "Get-LogEntries", "Get-LogSummary", "Get-RecentErrors") `
    -PowerShellVersion "7.0"
```

## Step 3 — the entry-point script: `Watch-Logs.ps1`

```powershell
# Watch-Logs.ps1
param(
    [Parameter(Mandatory)]
    [string]$LogDirectory,

    [string]$OutputPath = "log-summary.json",

    [int]$RecentErrorWindowMinutes = 120
)

$ErrorActionPreference = "Stop"

Import-Module (Join-Path $PSScriptRoot "LogMonitor/LogMonitor.psm1") -Force

try {
    $logFiles = Get-ChildItem -Path $LogDirectory -Filter "*.log" -File
    if ($logFiles.Count -eq 0) {
        Write-Warning "No .log files found in $LogDirectory"
        exit 0
    }

    $entries = $logFiles | Get-LogEntries

    $summary = $entries | Get-LogSummary
    $recentErrors = $entries | Get-RecentErrors -WithinMinutes $RecentErrorWindowMinutes

    Write-Output "=== Log Summary ==="
    Write-Output "Files scanned : $($logFiles.Count)"
    Write-Output "Total entries : $($summary.TotalEntries)"
    Write-Output "Error rate    : $($summary.ErrorRate)%"
    Write-Output ""
    Write-Output "By level:"
    $summary.ByLevel | Format-Table -AutoSize

    Write-Output "=== Errors in the last $RecentErrorWindowMinutes minutes (relative to newest entry) ==="
    if ($recentErrors) {
        $recentErrors | Select-Object Timestamp, Message | Format-Table -AutoSize
    } else {
        Write-Output "(none)"
    }

    $report = [ordered]@{
        GeneratedAt  = (Get-Date).ToString("s")
        FilesScanned = $logFiles.Count
        Summary      = $summary
        RecentErrors = @($recentErrors)
    }
    $report | ConvertTo-Json -Depth 5 | Set-Content -Path $OutputPath
    Write-Output "`nReport written to $OutputPath"
} catch {
    Write-Output "Log monitoring failed: $($_.Exception.Message)"
    exit 1
}
```

`$logFiles | Get-LogEntries` relies on `ValueFromPipelineByPropertyName`
(Module 01) — `Get-ChildItem` returns file objects with a `FullName`
property, and `Get-LogEntries`'s `-Path` parameter has an `-Alias
"FullName"`, so the pipeline binds them together without any manual
property extraction.

## Step 4 — Pester tests: `Tests/LogMonitor.Tests.ps1`

```powershell
BeforeAll {
    Import-Module "$PSScriptRoot/../LogMonitor/LogMonitor.psm1" -Force
}

Describe "ConvertFrom-LogLine" {
    It "parses a well-formed log line" {
        $result = ConvertFrom-LogLine -Line "2026-08-02 09:45:10 ERROR Failed to connect to db"

        $result.Level | Should -Be "ERROR"
        $result.Message | Should -Be "Failed to connect to db"
        $result.Timestamp | Should -BeOfType [datetime]
    }

    It "warns and emits nothing for a malformed line" {
        $warnings = @()
        $result = ConvertFrom-LogLine -Line "totally not a log line" `
            -WarningVariable warnings -WarningAction SilentlyContinue

        $result | Should -BeNullOrEmpty
        $warnings.Count | Should -Be 1
    }

    It "accepts pipeline input and processes multiple lines" {
        $lines = @(
            "2026-08-02 09:00:00 INFO Started"
            "2026-08-02 09:01:00 WARN Slow"
        )
        $results = $lines | ConvertFrom-LogLine
        $results.Count | Should -Be 2
        $results[1].Level | Should -Be "WARN"
    }
}

Describe "Get-LogSummary" {
    BeforeAll {
        $script:entries = @(
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 09:00"; Level = "INFO"; Message = "a" }
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 09:01"; Level = "ERROR"; Message = "b" }
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 09:02"; Level = "ERROR"; Message = "c" }
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 09:03"; Level = "INFO"; Message = "d" }
        )
    }

    It "counts total entries correctly" {
        $summary = $entries | Get-LogSummary
        $summary.TotalEntries | Should -Be 4
    }

    It "computes the error rate correctly" {
        $summary = $entries | Get-LogSummary
        $summary.ErrorRate | Should -Be 50.0
    }
}

Describe "Get-RecentErrors" {
    BeforeAll {
        $script:entries = @(
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 09:00"; Level = "ERROR"; Message = "old" }
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 10:50"; Level = "ERROR"; Message = "recent" }
            [pscustomobject]@{ Timestamp = [datetime]"2026-08-02 11:00"; Level = "INFO"; Message = "not an error" }
        )
    }

    It "only returns ERROR entries within the time window" {
        $recent = $entries | Get-RecentErrors -WithinMinutes 30
        $recent.Count | Should -Be 1
        $recent[0].Message | Should -Be "recent"
    }
}
```

Notice `ConvertFrom-LogLine`'s malformed-line test uses `-WarningVariable`
and `-WarningAction SilentlyContinue` together: it captures the warning
into `$warnings` for assertion *and* suppresses it from cluttering the test
output — a pattern worth reusing any time a test needs to assert that a
warning happened without printing it.

## Running it

```powershell
pwsh ./Watch-Logs.ps1 -LogDirectory ./logs -RecentErrorWindowMinutes 90
```

```text
WARNING: Skipping unparseable line: not a valid log line at all
=== Log Summary ===
Files scanned : 2
Total entries : 10
Error rate    : 30%

By level:

Name  Count
----  -----
INFO      6
ERROR     3
WARN      1

=== Errors in the last 90 minutes (relative to newest entry) ===

Timestamp              Message
---------              -------
02/08/2026 9:45:10 AM  Failed to connect to db
02/08/2026 10:15:03 AM Timeout contacting payment gateway
02/08/2026 10:20:00 AM Queue processing failed

Report written to log-summary.json
```

(Exact timestamp formatting depends on your machine's locale — the
underlying `[datetime]` values and filtering logic are what matters.)

Running the test suite:

```powershell
Invoke-Pester -Path ./Tests/LogMonitor.Tests.ps1 -Output Detailed
```

```text
Describing ConvertFrom-LogLine
  [+] parses a well-formed log line
  [+] warns and emits nothing for a malformed line
  [+] accepts pipeline input and processes multiple lines
Describing Get-LogSummary
  [+] counts total entries correctly
  [+] computes the error rate correctly
Describing Get-RecentErrors
  [+] only returns ERROR entries within the time window
Tests Passed: 6, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
```

## Where to take it from here

- Add a `-Since <datetime>` parameter to filter entries before summarizing,
  instead of always processing the whole file.
- Add a `Watch-LogFileLive` function using `Get-Content -Wait -Tail 0` to
  process new lines as they're appended in real time.
- Push alerts somewhere (email, webhook) when `ErrorRate` crosses a
  threshold, instead of only writing a report file.
- Package `LogMonitor` for the PowerShell Gallery (Module 08's
  `RequiredModules`/versioning conventions apply directly).

## Stretch goals

- Support multiple log-level formats (some systems use `DEBUG`/`TRACE` too)
  by making the pattern's level alternation a parameter instead of a
  hardcoded regex.
- Add a `Get-LogTrend` function that groups entries by hour and reports
  error counts per hour, so a spike is visible even in a summary.
- Write a Pester test that uses `Mock` on `Get-Content` to simulate a file
  read failure and assert `Get-LogEntries` reports it via `Write-Error`
  rather than throwing uncaught.
- Add a `-Format Csv` option to `Watch-Logs.ps1` that exports the parsed
  entries (not just the summary) using `Export-Csv`, for further analysis
  in a spreadsheet.
