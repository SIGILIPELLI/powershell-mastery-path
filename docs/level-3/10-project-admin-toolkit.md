# 10 · Project — Admin Toolkit Module

Time to bring Level 3 together into one real deliverable: an `AdminToolkit`
module that reports disk space and top CPU consumers, then assembles both
into a single JSON report — with [validated advanced
functions](01-advanced-functions-validation.md), a proper [Public/Private
module layout](04-building-reusable-modules.md), and [Pester tests that
mock across module boundaries](05-testing-advanced-mocking.md).

## Goal

1. `Get-DiskSpaceReport` — reports used/free space per filesystem drive,
   flagging any drive over a configurable warning threshold.
2. `Get-TopProcesses` — the top N processes by CPU time.
3. `New-AdminReport` — combines both into one object, writes it as JSON,
   and reports how many drives are in a warning state.
4. A private helper (`Format-ReportTimestamp`) shared internally, not
   exported.
5. A Pester suite covering all three public functions, mocking the
   underlying system cmdlets so tests don't depend on the actual disk
   layout or running processes of whatever machine runs them.

## Project layout

```text
AdminToolkit/
├── AdminToolkit.psd1
├── AdminToolkit.psm1
├── Public/
│   ├── Get-DiskSpaceReport.ps1
│   ├── Get-TopProcesses.ps1
│   └── New-AdminReport.ps1
├── Private/
│   └── Format-ReportTimestamp.ps1
└── Tests/
    └── AdminToolkit.Tests.ps1
```

## The public functions

```powershell
# Public/Get-DiskSpaceReport.ps1
function Get-DiskSpaceReport {
    [CmdletBinding()]
    param(
        [double]$WarningThresholdPercent = 80
    )
    $drives = Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Used -ne $null }
    foreach ($d in $drives) {
        $total = $d.Used + $d.Free
        if ($total -eq 0) { continue }
        $pctUsed = [math]::Round(($d.Used / $total) * 100, 1)
        [pscustomobject]@{
            Drive       = $d.Name
            UsedGB      = [math]::Round($d.Used / 1GB, 2)
            FreeGB      = [math]::Round($d.Free / 1GB, 2)
            PercentUsed = $pctUsed
            Status      = if ($pctUsed -ge $WarningThresholdPercent) { "Warning" } else { "OK" }
        }
    }
}
```

```powershell
# Public/Get-TopProcesses.ps1
function Get-TopProcesses {
    [CmdletBinding()]
    param([int]$Count = 5)
    Get-Process |
        Sort-Object -Property CPU -Descending |
        Select-Object -First $Count -Property Id, ProcessName,
            @{N='CPU_Seconds'; E={ [math]::Round($_.CPU, 2) }},
            @{N='MemoryMB'; E={ [math]::Round($_.WorkingSet64 / 1MB, 1) }}
}
```

```powershell
# Private/Format-ReportTimestamp.ps1
function Format-ReportTimestamp {
    (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss")
}
```

```powershell
# Public/New-AdminReport.ps1
function New-AdminReport {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$OutputPath,
        [double]$DiskWarningPercent = 80,
        [int]$TopProcessCount = 5
    )

    $disk  = Get-DiskSpaceReport -WarningThresholdPercent $DiskWarningPercent
    $procs = Get-TopProcesses -Count $TopProcessCount

    $report = [pscustomobject]@{
        GeneratedAt  = Format-ReportTimestamp
        Disks        = $disk
        TopProcesses = $procs
        WarningCount = @($disk | Where-Object Status -eq 'Warning').Count
    }

    $report | ConvertTo-Json -Depth 5 | Out-File -FilePath $OutputPath -Encoding utf8
    Write-Verbose "Report written to $OutputPath"
    return $report
}
```

`@(...)` around the `Where-Object` in `WarningCount` guards against a
single-item result: PowerShell unwraps a one-element pipeline result to a
scalar, so without `@()`, `.Count` on exactly one warning would throw
(scalars have no `.Count`) — a classic collection-vs-scalar trap. Wrapping
in `@()` forces it to stay an array regardless of how many items come
back, including zero.

## The loader and manifest

```powershell
# AdminToolkit.psm1
$here = $PSScriptRoot
foreach ($folder in 'Private', 'Public') {
    $path = Join-Path $here $folder
    if (Test-Path $path) {
        Get-ChildItem -Path $path -Filter '*.ps1' | ForEach-Object { . $_.FullName }
    }
}
$publicFunctions = Get-ChildItem -Path (Join-Path $here 'Public') -Filter '*.ps1' |
    ForEach-Object { $_.BaseName }
Export-ModuleMember -Function $publicFunctions
```

```powershell
# AdminToolkit.psd1
@{
    RootModule        = 'AdminToolkit.psm1'
    ModuleVersion     = '1.0.0'
    GUID              = 'c1d2e3f4-5566-7788-99aa-bbccddeeff00'
    Author            = 'Mastery Path'
    Description       = 'Small local admin reporting toolkit: disk space, top processes, JSON report.'
    PowerShellVersion = '5.1'
    FunctionsToExport = @('Get-DiskSpaceReport', 'Get-TopProcesses', 'New-AdminReport')
}
```

## Running it

```powershell
Import-Module ./AdminToolkit.psd1 -Force
Get-DiskSpaceReport | Format-Table
```

```text
Drive  UsedGB FreeGB PercentUsed Status
-----  ------ ------ ----------- ------
/     204.39   23.88        89.5 Warning
Temp  204.39   23.88        89.5 Warning
```

```powershell
Get-TopProcesses -Count 3 | Format-Table
```

```text
  Id ProcessName                     CPU_Seconds MemoryMB
  -- -----------                     ----------- --------
1260 zoom.us                            10020.11     76.5
1636 Google Chrome for Testing Helper    9941.95     62.5
1495 node                                3329.75     71.1
```

```powershell
$r = New-AdminReport -OutputPath ./report.json -Verbose
$r.WarningCount
```

```text
VERBOSE: Report written to ./report.json
2
```

```json
{
  "GeneratedAt": "2026-08-26T10:40:01",
  "Disks": [
    {
      "Drive": "/",
      "UsedGB": 204.39,
      "FreeGB": 23.88,
      "PercentUsed": 89.5,
      "Status": "Warning"
    },
    ...
  ]
}
```

## Tests, mocking across the module boundary

```powershell
# Tests/AdminToolkit.Tests.ps1
BeforeAll {
    Import-Module "$PSScriptRoot/../AdminToolkit.psd1" -Force
}

Describe "Get-DiskSpaceReport" {
    BeforeAll {
        Mock Get-PSDrive -ModuleName AdminToolkit {
            @(
                [pscustomobject]@{ Name = "C"; Used = 90GB; Free = 10GB },
                [pscustomobject]@{ Name = "D"; Used = 20GB; Free = 80GB }
            )
        }
    }

    It "flags a drive over the warning threshold" {
        $result = Get-DiskSpaceReport -WarningThresholdPercent 80
        ($result | Where-Object Drive -eq "C").Status | Should -Be "Warning"
    }

    It "does not flag a drive under the warning threshold" {
        $result = Get-DiskSpaceReport -WarningThresholdPercent 80
        ($result | Where-Object Drive -eq "D").Status | Should -Be "OK"
    }

    It "computes percent used correctly" {
        (Get-DiskSpaceReport | Where-Object Drive -eq "C").PercentUsed | Should -Be 90
    }
}

Describe "New-AdminReport" {
    BeforeAll {
        Mock Get-DiskSpaceReport -ModuleName AdminToolkit {
            @([pscustomobject]@{ Drive="C"; UsedGB=90; FreeGB=10; PercentUsed=90; Status="Warning" })
        }
        Mock Get-TopProcesses -ModuleName AdminToolkit {
            @([pscustomobject]@{ Id=1; ProcessName="test"; CPU_Seconds=1.0; MemoryMB=10.0 })
        }
    }

    It "counts warnings correctly" {
        $path = [System.IO.Path]::GetTempFileName()
        (New-AdminReport -OutputPath $path).WarningCount | Should -Be 1
        Remove-Item $path
    }

    It "writes valid JSON to the output path" {
        $path = [System.IO.Path]::GetTempFileName()
        New-AdminReport -OutputPath $path | Out-Null
        { Get-Content $path -Raw | ConvertFrom-Json } | Should -Not -Throw
        Remove-Item $path
    }
}
```

`Get-PSDrive` is mocked with `-ModuleName AdminToolkit` since
`Get-DiskSpaceReport` calls it internally — the same trap from module 05.
Mocking `Get-DiskSpaceReport` and `Get-TopProcesses` in the
`New-AdminReport` tests means those tests verify *composition* (does the
report correctly assemble and count what its dependencies return)
completely independent of whether the disk logic itself is correct — that
part is already covered by its own `Describe` block above.

```powershell
Invoke-Pester -Path ./Tests/AdminToolkit.Tests.ps1 -Output Detailed
```

```text
Describing Get-DiskSpaceReport
  [+] flags a drive over the warning threshold 113ms
  [+] does not flag a drive under the warning threshold 5ms
  [+] computes percent used correctly 11ms
Describing New-AdminReport
  [+] counts warnings correctly 28ms
  [+] writes valid JSON to the output path 48ms
Tests Passed: 5, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
```

## Stretch goals

- Add a `Get-ServiceStatusReport` public function checking a list of
  service names against `Get-Service`, include it in `New-AdminReport`,
  and mock `Get-Service` in a new test `Describe` block.
- Add an `-EmailTo` parameter to `New-AdminReport` that, when a warning
  exists, calls `Send-MailMessage` (mocked in tests) with the report
  summarized in the body.
- Package the module with a `ScheduledTask`/cron entry (module 09) that
  runs `New-AdminReport` nightly and only alerts when `WarningCount -gt 0`.
- Add `ValidateScript` to `-OutputPath` in `New-AdminReport` confirming the
  parent directory exists before attempting the write, with a clear error
  message if not.
