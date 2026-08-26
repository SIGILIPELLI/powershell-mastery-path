# 07 · Building PowerShell-based DevOps Tools

Everything so far has built toward this: a script meant to be run by
other people (or other pipelines) as a real CLI tool, not read as source
code. That means parameter sets that make invalid combinations impossible,
predictable exit codes, and structured logging — the difference between a
script and a tool.

## Parameter sets: making invalid combinations unrepresentable

```powershell
[CmdletBinding(DefaultParameterSetName = 'Status')]
param(
    [Parameter(ParameterSetName = 'Deploy', Mandatory)]
    [switch]$Deploy,

    [Parameter(ParameterSetName = 'Deploy', Mandatory)]
    [Parameter(ParameterSetName = 'Rollback', Mandatory)]
    [string]$Environment,

    [Parameter(ParameterSetName = 'Rollback', Mandatory)]
    [switch]$Rollback,

    [Parameter(ParameterSetName = 'Status')]
    [switch]$Status
)

switch ($PSCmdlet.ParameterSetName) {
    'Deploy'   { "Deploying to $Environment..." }
    'Rollback' { "Rolling back $Environment..." }
    'Status'   { "Checking status of all environments..." }
}
```

```powershell
./deploycli.ps1
```
```text
Checking status of all environments...
```

```powershell
./deploycli.ps1 -Deploy -Environment prod
```
```text
Deploying to prod...
```

```powershell
./deploycli.ps1 -Rollback -Environment staging
```
```text
Rolling back staging...
```

`$Environment` belongs to *both* the `Deploy` and `Rollback` sets (two
`[Parameter(...)]` attributes on one parameter), while `-Deploy`,
`-Rollback`, and `-Status` each anchor their own set — PowerShell figures
out which set is active from which switches were passed, and
`$PSCmdlet.ParameterSetName` tells you which one won. Crucially, this
makes `-Deploy -Rollback` together a **parse-time error**, not something
your code has to detect and reject manually — the invalid combination
literally cannot bind.

Passing `-Deploy` without the also-mandatory `-Environment` triggers
PowerShell's normal mandatory-parameter prompt:

```powershell
./deploycli.ps1 -Deploy
```
```text
cmdlet deploycli.ps1 at command pipeline position 1
Supply values for the following parameters:
Environment:
```

That interactive prompt is exactly why every unattended/CI invocation of
a tool like this must pass every mandatory parameter explicitly — a
missing one doesn't fail cleanly by default, it **hangs waiting for
input that will never come** in a non-interactive context. (Redirecting
empty input, as CI often does implicitly, turns that hang into the
"missing mandatory parameters" error instead — better, but still not as
clean as never triggering the prompt at all.)

## Exit codes and structured logging

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [ValidateSet('dev','staging','prod')]
    [string]$Environment,

    [switch]$DryRun
)

$ErrorActionPreference = 'Stop'

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $line = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') [$Level] $Message"
    Write-Output $line
}

try {
    Write-Log "Starting deploy to $Environment (DryRun=$DryRun)"

    if ($DryRun) {
        Write-Log "Dry run - no changes made"
        exit 0
    }

    if ($Environment -eq 'staging') {
        throw "Simulated failure: staging artifact not found"
    }

    Write-Log "Deploy to $Environment succeeded"
    exit 0
} catch {
    Write-Log "Deploy failed: $($_.Exception.Message)" -Level ERROR
    exit 1
}
```

```powershell
./deploy-tool.ps1 -Environment dev -DryRun
```
```text
2026-08-26 11:04:33 [INFO] Starting deploy to dev (DryRun=True)
2026-08-26 11:04:33 [INFO] Dry run - no changes made
```

```powershell
./deploy-tool.ps1 -Environment staging
```
```text
2026-08-26 11:04:34 [INFO] Starting deploy to staging (DryRun=False)
2026-08-26 11:04:34 [ERROR] Deploy failed: Simulated failure: staging artifact not found
```
Exit code: `1`.

Three habits that turn a script into something automation can rely on:

- **`$ErrorActionPreference = 'Stop'`** at the top means a non-terminating
  error from a cmdlet inside the `try` still gets caught, rather than
  printing a warning and continuing past a real failure — the single most
  common reason a "successful" CI step actually did nothing.
- **`Write-Log` with a level and timestamp on every line** — the exact
  same pattern module 09 (Logging & Observability) builds out further,
  but the core idea starts here: consistent structure means downstream
  tooling (log aggregation, `Select-String`, alerting) can parse it
  reliably instead of scraping free-form text.
- **Explicit `exit 0` / `exit 1`** — never rely on PowerShell's own
  fall-through exit code for a tool meant to be invoked by another
  system; state the outcome as a number every time, on every code path.

## Designing for the caller, not just yourself

A DevOps tool gets invoked by people who didn't write it and by pipelines
that can't ask it questions — a few conventions make that much smoother:

- **`-DryRun` / `-WhatIf`** on anything destructive, so a caller can
  verify intent before committing to it (this mirrors `SupportsShouldProcess`
  from earlier modules, but a simple `-DryRun` switch works fine for a
  standalone tool too).
- **`-Verbose` for detail, plain output for the result** — the default,
  non-verbose run should be quiet and just report success/failure; put
  step-by-step detail behind `Write-Verbose` so both a human debugging
  interactively and a quiet CI log get what they each actually want.
- **Consistent noun-verb naming across your own tools** — `deploy-tool.ps1
  -Environment prod`, `rollback-tool.ps1 -Environment prod`,
  `status-tool.ps1` reads far more predictably as a family than three
  scripts with unrelated argument conventions.

## Cheat sheet

| Feature | Purpose |
|---|---|
| Multiple `[Parameter(ParameterSetName=...)]` per param | share a parameter across sets |
| `$PSCmdlet.ParameterSetName` | detect which set actually bound |
| Mandatory param missing, no input redirected | hangs on a prompt — always pass everything explicitly in automation |
| `$ErrorActionPreference = 'Stop'` | non-terminating errors become catchable |
| `Write-Log` with level + timestamp | structured, parseable output |
| Explicit `exit 0` / `exit 1` | reliable outcome signal for any caller |
| `-DryRun` / `-WhatIf` | let callers verify intent before committing |
| `-Verbose` for detail, quiet default output | serves both interactive and CI use |

## Exercise

Build a `db-tool.ps1` with three parameter sets — `Backup` (requires
`-Database`), `Restore` (requires `-Database` and `-BackupFile`), and
`List` (no extra parameters, the default set) — each writing structured
log lines via a shared `Write-Log` function and exiting with the correct
code on simulated success/failure. Confirm `$PSCmdlet.ParameterSetName`
correctly resolves for each combination, and confirm an invalid
combination (like `-Database` and `-BackupFile` together with no
`-Restore` context) is rejected at parse time rather than needing manual
validation in the body.
