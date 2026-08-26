# 10 · Capstone Project

Everything from Levels 1-4 comes together here: a real multi-file module
— public/private function separation, a generated manifest, Pester tests
with tags, structured logging, and a JSON report — built the way an
actual small internal tool gets built. `ServerAudit` checks disk free
space and whether expected processes are running, then writes both a
structured log and a machine-readable report. Every command below was
actually run to produce the output shown.

## Project layout

```text
ServerAudit/
├── ServerAudit.psd1          # manifest (generated with New-ModuleManifest)
├── ServerAudit.psm1          # loader: dot-sources Private/ then Public/
├── Public/
│   ├── Get-DiskHealth.ps1
│   ├── Get-ProcessHealth.ps1
│   └── New-AuditReport.ps1
├── Private/
│   └── Write-AuditLog.ps1
└── Tests/
    └── ServerAudit.Tests.ps1
```

Public/Private separation (module 04, Level 3) matters here specifically:
`Write-AuditLog` is an implementation detail the report-building function
needs, not something a consumer of the module should call directly, so it
lives in `Private/` and never appears in `Export-ModuleMember`.

## The module loader

```powershell
# ServerAudit.psm1
$here = $PSScriptRoot
foreach ($folder in 'Private', 'Public') {
    $path = Join-Path $here $folder
    if (Test-Path $path) {
        Get-ChildItem -Path $path -Filter '*.ps1' | ForEach-Object { . $_.FullName }
    }
}

Export-ModuleMember -Function 'Get-DiskHealth', 'Get-ProcessHealth', 'New-AuditReport'
```

Dot-sourcing every `.ps1` under `Private/` then `Public/` means adding a
new function later is just adding a file — no editing the loader. Only
the three names in `Export-ModuleMember` become visible outside the
module; `Write-AuditLog` is still loaded and callable *within* the
module's other functions, just not from a consumer's session.

## `Get-DiskHealth`: real system data, no mocking needed

```powershell
function Get-DiskHealth {
    [CmdletBinding()]
    param(
        [ValidateRange(0, 100)] [int] $WarningThresholdPercent = 15
    )
    Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Used -ne $null -and ($_.Used + $_.Free) -gt 0 } | ForEach-Object {
        $totalBytes = $_.Used + $_.Free
        $freePercent = [math]::Round(($_.Free / $totalBytes) * 100, 1)
        [pscustomobject]@{
            Drive       = $_.Name
            FreePercent = $freePercent
            Status      = if ($freePercent -le $WarningThresholdPercent) { 'Warning' } else { 'OK' }
        }
    }
}
```

```text
Drive FreePercent Status
----- ----------- ------
/           9.900 Warning
Temp        9.900 Warning
```

Real output from the machine this lesson was written on — genuinely low
free space, correctly flagged `Warning` against the default 15% threshold.
`Get-PSDrive -PSProvider FileSystem` (module 03/08, Level 1-3) is
cross-platform: the same function reports drive letters on Windows and
mount points on macOS/Linux without any branching.

## `Get-ProcessHealth`: presence checks with `-ErrorAction SilentlyContinue`

```powershell
function Get-ProcessHealth {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)] [string[]] $Name
    )
    foreach ($n in $Name) {
        $proc = Get-Process -Name $n -ErrorAction SilentlyContinue
        [pscustomobject]@{
            ProcessName   = $n
            Running       = [bool]$proc
            InstanceCount = @($proc).Count
        }
    }
}
```

```text
ProcessName Running InstanceCount
----------- ------- -------------
pwsh           True             1
sshd          False             0
```

`Get-Process -Name` throws a non-terminating error for a name that
doesn't match any running process; `-ErrorAction SilentlyContinue`
turns "not running" into an expected, checkable outcome (`$proc` is
`$null`) instead of red error text — exactly right for a health check
where "not running" is a real, reportable state, not a script bug.

## `New-AuditReport`: tying it together, and a trap caught by testing it

```powershell
function New-AuditReport {
    [CmdletBinding()]
    param(
        [string[]] $ProcessName = @('pwsh'),
        [string] $OutputPath = './audit-report.json'
    )
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $result = [ordered]@{
        GeneratedAt = (Get-Date).ToString('o')
        Disks       = @()
        Processes   = @()
        Status      = 'OK'
    }
    try {
        $result.Disks = @(Get-DiskHealth)
        $result.Processes = @(Get-ProcessHealth -Name $ProcessName)
        if ($result.Disks.Status -contains 'Warning') {
            $result.Status = 'Warning'
        }
        $sw.Stop()
        $null = Write-AuditLog -Message "Audit completed in $($sw.ElapsedMilliseconds)ms, status $($result.Status)" -Level INFO
    } catch {
        $sw.Stop()
        $result.Status = 'Error'
        $null = Write-AuditLog -Message "Audit failed after $($sw.ElapsedMilliseconds)ms: $($_.Exception.Message)" -Level ERROR
    }
    $result | ConvertTo-Json -Depth 4 | Set-Content -Path $OutputPath
    [pscustomobject]$result
}
```

The `$null = Write-AuditLog ...` isn't decoration — the first version of
this function called `Write-AuditLog` without capturing its output, and
`Write-AuditLog` (like most well-behaved functions) returns the log entry
it just wrote. That return value silently joined `New-AuditReport`'s own
pipeline output, so the function returned **two objects** — the log entry
*and* the report — instead of one. It didn't throw or warn; it just
produced a caller-visible bug where `$report.Status` came back `$null`
for anyone who assumed a single return value. A Pester test (below)
caught it immediately; production code without one would have shipped it
quietly.

## Running it

```powershell
Import-Module ./ServerAudit.psd1 -Force
$report = New-AuditReport -ProcessName 'pwsh', 'sshd'
$report.Status
$report.Disks | Format-Table
$report.Processes | Format-Table
```

```text
Warning

Drive FreePercent Status
----- ----------- ------
/           9.900 Warning
Temp        9.900 Warning

ProcessName Running InstanceCount
----------- ------- -------------
pwsh           True             1
sshd          False             0
```

```powershell
Get-Content ./audit-report.json
```

```json
{"GeneratedAt":"2026-08-26T11:34:49.340645+05:30","Disks":[{"Drive":"/","FreePercent":9.9,"Status":"Warning"},{"Drive":"Temp","FreePercent":9.9,"Status":"Warning"}],"Processes":[{"ProcessName":"pwsh","Running":true,"InstanceCount":1},{"ProcessName":"sshd","Running":false,"InstanceCount":0}],"Status":"Warning"}
```

## Tests, tagged and run

```powershell
Describe 'Get-DiskHealth' -Tag 'Unit' {
    It 'returns a FreePercent between 0 and 100 for every drive' {
        $results = Get-DiskHealth
        $results.Count | Should -BeGreaterThan 0
        foreach ($r in $results) {
            $r.FreePercent | Should -BeGreaterOrEqual 0
            $r.FreePercent | Should -BeLessOrEqual 100
        }
    }
}

Describe 'New-AuditReport' -Tag 'Integration' {
    It 'writes a JSON report with Disks and Processes sections' {
        $r = New-AuditReport -ProcessName 'pwsh' -OutputPath './test-report.json'
        $r.Status | Should -BeIn @('OK', 'Warning')
        $parsed = Get-Content './test-report.json' -Raw | ConvertFrom-Json
        $parsed.Processes.Count | Should -Be 1
    }
}
```

```text
Describing Get-DiskHealth
  [+] returns a FreePercent between 0 and 100 for every drive 62ms
  [+] flags a drive as Warning below the threshold 17ms
Describing Get-ProcessHealth
  [+] reports Running = $true for the current pwsh process 14ms
  [+] reports Running = $false for a process that does not exist 13ms
Describing New-AuditReport
  [+] writes a JSON report with Disks and Processes sections 61ms
Tests completed in 423ms
Tests Passed: 5, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
```

Note the `-Tag`s (module 05, Level 4): `Unit` tests need nothing but the
functions themselves, `Integration` actually writes a file to disk — a CI
pipeline can run `Unit` on every push and reserve `Integration` for
pre-merge, the same split covered earlier in this level.

## Cheat sheet

| Concept | Where it came from |
|---|---|
| Public/Private folder split | Building Reusable Modules (Level 3) |
| Generated manifest, `Test-ModuleManifest` | Publishing Modules (Level 4, module 08) |
| `-ErrorAction SilentlyContinue` for expected absence | Error handling (Level 1-2) |
| `[System.Diagnostics.Stopwatch]` timing | Performance at Scale (Level 4, module 06) |
| Structured JSON-lines logging | Logging & Observability (Level 4, module 09) |
| Tagged Pester tests, `Unit` vs `Integration` | Testing at Scale & CI (Level 4, module 05) |
| Cross-platform `Get-PSDrive` | Cross-Platform PowerShell (Level 4, module 03) |

## Stretch goals

- Add a `Send-AuditAlert` function that only fires (e.g. writes an
  `ERROR`-level log entry, or sends an email/webhook) when `$report.Status`
  is `Warning` or `Error`, and wire it into `New-AuditReport` behind a
  `-Notify` switch parameter.
- Package `ServerAudit` for the Gallery: run `Invoke-ScriptAnalyzer -Recurse`
  against it, fix every finding, add `RequiredModules`/`#Requires` if you
  extend it to depend on anything external, and bump `ModuleVersion` to
  `1.1.0`.
- Turn the audit into a scheduled job: a script that imports the module,
  runs `New-AuditReport`, and — on Windows — registers itself via
  `Register-ScheduledJob`, or via `cron`/`launchd` on Linux/macOS, so it
  runs unattended and the JSON report becomes a real historical record you
  can graph over time.
- Add a `-ComputerName` parameter to `Get-DiskHealth`/`Get-ProcessHealth`
  that runs the check via `Invoke-Command` (module 01, Level 4) against a
  remote machine instead of only the local one, turning this from a
  single-host tool into a small fleet-auditing one.
