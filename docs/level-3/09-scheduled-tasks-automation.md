# 09 · Scheduled Tasks & Automation

A script you run by hand isn't automation yet — it's automation once
something else triggers it on a schedule, without you watching. This
module covers Windows Task Scheduler's PowerShell cmdlets, cron on
Linux/macOS, and the automation patterns (retries, background jobs) that
matter regardless of which scheduler is running your script.

!!! note "Platform scope"
    `ScheduledTask` cmdlets (`Register-ScheduledTask`, etc.) are
    Windows-only and weren't available to run on this machine (macOS) —
    that section is documented for accuracy but not executed. Everything
    else in this module — `Start-Job`, the retry pattern, and cron — ran
    for real, shown with actual output below.

## Windows: `Register-ScheduledTask`

```powershell
$action = New-ScheduledTaskAction -Execute "pwsh.exe" `
    -Argument "-File C:\Scripts\Nightly-Backup.ps1"

$trigger = New-ScheduledTaskTrigger -Daily -At "2:00AM"

$settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 5)

Register-ScheduledTask -TaskName "NightlyBackup" `
    -Action $action -Trigger $trigger -Settings $settings `
    -User "SYSTEM" -RunLevel Highest
```

Four pieces every scheduled task needs: an **action** (what to run), a
**trigger** (when), **settings** (retry/timeout behavior), and the
**principal** (which account it runs as — `SYSTEM` here, so it works
whether or not anyone's logged in). `-StartWhenAvailable` matters for
laptops: without it, a trigger that fires while the machine is asleep is
simply missed, not deferred.

```powershell
Get-ScheduledTask -TaskName "NightlyBackup" | Get-ScheduledTaskInfo
```

```text
LastRunTime        : 8/26/2026 2:00:03 AM
LastTaskResult      : 0
NextRunTime        : 8/27/2026 2:00:00 AM
NumberOfMissedRuns : 0
```

`LastTaskResult : 0` means success — any nonzero value is the exit code
the script/process returned, which is why a well-behaved scheduled script
should always `exit` with a meaningful nonzero code on failure rather
than letting an unhandled exception produce an opaque one.

```powershell
Disable-ScheduledTask -TaskName "NightlyBackup"
Unregister-ScheduledTask -TaskName "NightlyBackup" -Confirm:$false
```

## Cross-platform: cron

On Linux and macOS, `pwsh` scripts are scheduled the normal Unix way —
PowerShell doesn't need to own the scheduler, just be the interpreter
cron invokes:

```text
# crontab -e
0 2 * * * /usr/local/bin/pwsh -File /scripts/nightly-backup.ps1 >> /var/log/backup.log 2>&1
```

The trap here isn't PowerShell-specific but bites PowerShell users
constantly: cron runs with a minimal environment (no interactive
`$PROFILE`, often a different `$PATH`), so a script that "just runs" from
your terminal can fail silently under cron because it depended on
something your interactive shell set up. Always test with
`pwsh -NoProfile -File script.ps1` to reproduce cron's stripped-down
environment before trusting a crontab entry.

## `Start-Job`: background execution inside a running session

```powershell
$job = Start-Job -ScriptBlock {
    Start-Sleep -Seconds 1
    "Job finished at $(Get-Date -Format 'HH:mm:ss')"
}
Write-Output "Job started, State: $($job.State)"
$job | Wait-Job | Out-Null
Receive-Job -Job $job
Remove-Job -Job $job
```

```text
Job started, State: Running
Job finished at 10:38:42
```

`Start-Job` runs the scriptblock in a **separate process**, which is why
`$job.State` reads `Running` immediately — the parent script doesn't
block. `Wait-Job` blocks until it's done, `Receive-Job` pulls back its
output, and `Remove-Job` cleans up the job object (jobs aren't
automatically garbage collected — leaving hundreds of finished jobs
un-removed across a long-running scheduler process is a slow, easy-to-miss
memory leak).

```powershell
$job2 = Start-Job -ScriptBlock {
    param($Name)
    "Hello, $Name"
} -ArgumentList "Automation"
$job2 | Wait-Job | Out-Null
Receive-Job -Job $job2
```

```text
Hello, Automation
```

Because the job runs in a separate process, it does **not** inherit
variables from your current session automatically — `-ArgumentList` is
how you pass data in explicitly; a scriptblock referencing an outer
`$variable` without `-ArgumentList` or `$using:variable` will see `$null`
inside the job.

## Retry with backoff: the pattern every scheduled script needs

Scheduled automation runs unattended, often against flaky external
dependencies (a network share, an API) — a single transient failure
shouldn't fail the whole run:

```powershell
function Invoke-WithRetry {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [scriptblock]$ScriptBlock,
        [int]$MaxAttempts = 3,
        [int]$DelaySeconds = 1
    )
    $attempt = 0
    while ($true) {
        $attempt++
        try {
            return & $ScriptBlock
        } catch {
            if ($attempt -ge $MaxAttempts) {
                throw
            }
            Write-Warning "Attempt $attempt failed: $($_.Exception.Message). Retrying in $DelaySeconds s..."
            Start-Sleep -Seconds $DelaySeconds
            $DelaySeconds *= 2
        }
    }
}

$script:tries = 0
Invoke-WithRetry -MaxAttempts 4 -DelaySeconds 1 -ScriptBlock {
    $script:tries++
    if ($script:tries -lt 3) { throw "Simulated transient failure #$script:tries" }
    "Succeeded on attempt $script:tries"
}
```

```text
WARNING: Attempt 1 failed: Simulated transient failure #1. Retrying in 1 s...
WARNING: Attempt 2 failed: Simulated transient failure #2. Retrying in 2 s...
Succeeded on attempt 3
```

Doubling `$DelaySeconds` each retry (exponential backoff) matters for
anything hitting a remote service — retrying instantly and repeatedly
against a struggling API makes the underlying problem worse, not better.
`throw` with no argument inside the final `catch` re-raises the original
exception rather than a generic one, preserving the real error for
whatever logging catches it upstream.

## The trap: a scheduled script that "works" interactively but not unattended

Three things silently differ between running a script yourself and a
scheduler running it:

- **No interactive prompts** — a script that calls `Read-Host` or hits a
  confirmation prompt (`Remove-Item` without `-Confirm:$false` under
  `$ConfirmPreference`) will simply hang forever with no one there to
  answer it.
- **Different working directory** — scheduled tasks and cron jobs often
  start in a directory other than the script's own; always build paths
  from `$PSScriptRoot`, never assume the current directory.
- **No profile, minimal environment** — as above; test with `-NoProfile`.

## Cheat sheet

| Tool | Platform | Purpose |
|---|---|---|
| `Register-ScheduledTask` + Action/Trigger/Settings | Windows | schedule a recurring script |
| `Get-ScheduledTaskInfo` | Windows | last run time/result, next run time |
| `crontab -e` | Linux/macOS | schedule `pwsh -File ...` |
| `Start-Job` / `Wait-Job` / `Receive-Job` / `Remove-Job` | cross-platform | background execution in a separate process |
| `-ArgumentList` / `$using:var` | cross-platform | pass data into a job (no automatic variable inheritance) |
| Retry-with-backoff pattern | cross-platform | survive transient failures unattended |
| `$PSScriptRoot` for all paths | cross-platform | scheduler working directory isn't guaranteed |

## Exercise

Write a script `Invoke-ScheduledSync.ps1` meant to run under cron/Task
Scheduler: it should use `$PSScriptRoot` for all file paths, wrap its main
work in `Invoke-WithRetry` (3 attempts, 2-second initial backoff), log
success/failure with a timestamp to a file next to the script, and `exit 1`
on final failure so the scheduler's own "last result" reflects it. Test it
both interactively and with `pwsh -NoProfile -File` to confirm it behaves
identically both ways.
