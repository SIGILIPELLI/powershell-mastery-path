# 10 · Project — System Info Reporter

Time to combine everything from Level 1 — variables, control flow,
functions, the object pipeline, arrays/hashtables, files, error handling, and
a module — into one working script: a **system info reporter** that gathers
basic facts about the machine it's run on and writes a clean report to both
the console and a JSON file.

## Goal

Build `SystemReport.ps1` that:

1. Collects OS, PowerShell version, CPU/memory, and top processes by memory.
2. Formats the results as a friendly console report.
3. Handles missing/unavailable data gracefully (some info differs across
   Windows/macOS/Linux).
4. Exports the full report as JSON for later use (e.g. by a monitoring
   dashboard).
5. Is organized as a small module (`SystemInfo.psm1`) plus a thin
   entry-point script that calls it — mirroring how real production
   automation is structured.

## Step 1 — the module: `SystemInfo.psm1`

```powershell
# SystemInfo.psm1

function Get-BasicSystemInfo {
    [CmdletBinding()]
    param()

    try {
        $os = [System.Runtime.InteropServices.RuntimeInformation]::OSDescription
    } catch {
        $os = "Unknown"
    }

    [pscustomobject]@{
        OSDescription     = $os
        PSVersion         = $PSVersionTable.PSVersion.ToString()
        MachineName       = $env:COMPUTERNAME ? $env:COMPUTERNAME : (hostname)
        ProcessorCount    = [Environment]::ProcessorCount
        GeneratedAt       = (Get-Date).ToString("s")
    }
}

function Get-TopMemoryProcesses {
    [CmdletBinding()]
    param(
        [int]$Top = 5
    )

    Get-Process |
        Sort-Object WorkingSet64 -Descending |
        Select-Object -First $Top -Property ProcessName, Id,
            @{Name = "MemoryMB"; Expression = { [math]::Round($_.WorkingSet64 / 1MB, 1) }}
}

function Get-SystemReport {
    [CmdletBinding()]
    param(
        [int]$TopProcessCount = 5
    )

    $report = [ordered]@{
        SystemInfo    = Get-BasicSystemInfo
        TopProcesses  = @(Get-TopMemoryProcesses -Top $TopProcessCount)
    }

    return $report
}

Export-ModuleMember -Function Get-BasicSystemInfo, Get-TopMemoryProcesses, Get-SystemReport
```

A few things worth noting:

- `[pscustomobject]@{...}` builds a lightweight, ordered custom object —
  the idiomatic PowerShell way to return structured data instead of a plain
  hashtable, because it prints as a clean table and keeps property order.
- The `?:`-style ternary (`$env:COMPUTERNAME ? ... : ...`) is a PowerShell
  7+ feature; on `pwsh` 7.0+ it works out of the box.
- Wrapping OS detection in `try/catch` means the script degrades gracefully
  instead of crashing on a platform where a given API isn't available.

## Step 2 — the entry-point script: `SystemReport.ps1`

```powershell
# SystemReport.ps1
param(
    [string]$OutputPath = "system-report.json",
    [int]$TopProcessCount = 5
)

Import-Module (Join-Path $PSScriptRoot "SystemInfo.psm1") -Force

try {
    $report = Get-SystemReport -TopProcessCount $TopProcessCount
} catch {
    Write-Output "Failed to build system report: $($_.Exception.Message)"
    exit 1
}

Write-Output "=== System Info ==="
$report.SystemInfo | Format-List

Write-Output "`n=== Top $TopProcessCount Processes by Memory ==="
$report.TopProcesses | Format-Table -AutoSize

try {
    $report | ConvertTo-Json -Depth 4 | Set-Content -Path $OutputPath
    Write-Output "`nReport written to $OutputPath"
} catch {
    Write-Output "Failed to write report file: $($_.Exception.Message)"
    exit 1
}
```

`$PSScriptRoot` is an automatic variable holding the directory the currently
running script lives in — using it to build the module path means the
script works no matter what directory you invoke it from.

## Step 3 — running it

```powershell
pwsh ./SystemReport.ps1
```

```text
=== System Info ===

OSDescription  : macOS 14.5
PSVersion      : 7.4.1
MachineName    : adas-macbook
ProcessorCount : 10
GeneratedAt    : 2026-07-20T09:15:03

=== Top 5 Processes by Memory ===

ProcessName     Id MemoryMB
-----------     -- --------
chrome        41022    892.3
pwsh           1234    145.7
Finder         5678     88.9
Dock           9012     52.1
Spotlight      3344     47.6

Report written to system-report.json
```

```powershell
pwsh ./SystemReport.ps1 -OutputPath "reports/latest.json" -TopProcessCount 10
```

## Step 4 — verifying the JSON output

```powershell
Get-Content ./system-report.json | ConvertFrom-Json | Select-Object -ExpandProperty SystemInfo
```

```text
OSDescription  : macOS 14.5
PSVersion      : 7.4.1
MachineName    : adas-macbook
ProcessorCount : 10
GeneratedAt    : 2026-07-20T09:15:03
```

## Where to take it from here

- Add a `-Format` parameter that switches between console output, JSON, and
  CSV using `switch`.
- Add disk-space info with `Get-PSDrive -PSProvider FileSystem`.
- Wrap the whole thing to run on a schedule (Level 2/3 territory —
  `Scheduled Tasks & Automation`).
- Add Pester tests for `Get-BasicSystemInfo` and `Get-TopMemoryProcesses`
  (Level 2's testing module).

## Exercise

Extend `SystemInfo.psm1` with a new function `Get-DiskSummary` that uses
`Get-PSDrive -PSProvider FileSystem` to report each drive's name, used space,
and free space (in GB, rounded to 1 decimal), export it from the module, and
add its output as a third section (`DiskInfo`) in both the console report
and the JSON file produced by `SystemReport.ps1`.
