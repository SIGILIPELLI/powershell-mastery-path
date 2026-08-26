# 01 · Advanced Remoting & Background Jobs

Level 3 introduced `Start-Job` for basic background execution. At scale —
dozens of tasks, or genuine remote machines — you need lighter-weight
parallelism than a full separate process per job, and you need to
understand exactly what state does and doesn't cross into a parallel
execution context.

!!! note "PSSession / WinRM scope"
    `New-PSSession`/`Invoke-Command -ComputerName` against a real remote
    Windows host need WinRM configured on that host, which this machine
    doesn't have a target for — that section below is accurate but not
    executed live. `Start-ThreadJob` and `ForEach-Object -Parallel`, which
    make up most of this module, ran for real on this machine and their
    output is genuine.

## `Start-ThreadJob`: parallelism without a new process per job

```powershell
$sw = [System.Diagnostics.Stopwatch]::StartNew()
$jobs = 1..4 | ForEach-Object {
    Start-ThreadJob -ScriptBlock {
        param($n)
        Start-Sleep -Seconds 1
        "Task $n done"
    } -ArgumentList $_
}
$jobs | Wait-Job | Receive-Job
$sw.Stop()
"Total elapsed: $([math]::Round($sw.Elapsed.TotalSeconds,1))s"
$jobs | Remove-Job
```

```text
Task 1 done
Task 2 done
Task 3 done
Task 4 done
Total elapsed: 1.6s
```

Four one-second tasks finished in ~1.6s, not 4s — they ran concurrently.
`Start-ThreadJob` (built into PowerShell 7) uses a thread inside the
current process instead of spinning up a whole new `pwsh` process the way
`Start-Job` does, which makes it dramatically cheaper to create many of —
useful when you need tens or hundreds of small concurrent tasks rather
than a handful of heavy ones.

## `ForEach-Object -Parallel`: the simplest parallel loop

```powershell
1..4 | ForEach-Object -Parallel {
    Start-Sleep -Seconds 1
    "Parallel task $_ done"
} -ThrottleLimit 4
```

```text
Parallel task 1 done
Parallel task 2 done
Parallel task 3 done
Parallel task 4 done
```

`-ThrottleLimit` caps how many run at once (default is 5) — set it
deliberately rather than leaving the default when the work is
CPU-intensive or hits a rate-limited external API; too high a limit turns
"faster" into "everything contends for the same resource and gets
slower."

## The trap: outer variables don't cross into a parallel scriptblock

```powershell
$multiplier = 10
1..3 | ForEach-Object -Parallel {
    $_ * $multiplier
}
```

```text
0
0
0
```

Each parallel iteration runs in its own **isolated runspace** — it does
not share the calling scope, so `$multiplier` inside the block is
`$null`, and `$null * 3` is `0`, not an error, which makes this bug easy
to miss. The fix is `$using:`, which explicitly copies a value from the
caller's scope into the runspace:

```powershell
1..3 | ForEach-Object -Parallel {
    $_ * $using:multiplier
}
```

```text
10
20
30
```

`$using:` also works the same way inside `Invoke-Command` scriptblocks
sent to a remote session — same underlying idea, cross-runspace variable
capture.

## The trap: parallel runspaces can't share a plain array or hashtable

```powershell
$results = [System.Collections.Concurrent.ConcurrentBag[int]]::new()
1..5 | ForEach-Object -Parallel {
    ($using:results).Add($_ * $_)
}
($results | Sort-Object) -join ","
```

```text
1,4,9,16,25
```

A regular `@()` array or `[List[T]]` is **not thread-safe** — multiple
runspaces calling `.Add()` on the same plain list concurrently can
corrupt it or silently drop items, because normal collections have no
protection against two threads mutating them at the same instant.
`System.Collections.Concurrent.ConcurrentBag[T]` (and its siblings
`ConcurrentQueue`, `ConcurrentDictionary`) are specifically designed to be
safely written to from multiple threads at once — reach for one whenever
parallel work needs to accumulate results into a shared collection.

## `Invoke-Command` against a real remote machine

```powershell
$session = New-PSSession -ComputerName "Server01" -Credential (Get-Credential)

Invoke-Command -Session $session -ScriptBlock {
    Get-Service | Where-Object Status -eq "Running" | Measure-Object
}

Invoke-Command -Session $session -ScriptBlock {
    param($ServiceName)
    Restart-Service -Name $ServiceName -Force
} -ArgumentList "Spooler"

Remove-PSSession $session
```

`New-PSSession` opens a persistent connection you can reuse across
multiple `Invoke-Command` calls (each carrying state — variables set in
one call are visible in the next, within that session) — cheaper than
`Invoke-Command -ComputerName` on its own, which opens and tears down a
new connection per call. Always `Remove-PSSession` when done; leaked open
sessions accumulate on the remote machine and eventually hit its
connection limit.

```powershell
# Fan out to several machines, same command, one round trip
Invoke-Command -ComputerName "Web01", "Web02", "Web03" -ScriptBlock {
    Get-CimInstance Win32_OperatingSystem | Select-Object LastBootUpTime
}
```

Passing an array to `-ComputerName` runs the scriptblock on every machine
**in parallel** by default (throttled at 32 concurrent by default,
adjustable with `-ThrottleLimit`) and returns all results tagged with
`PSComputerName` — a single command instead of a manual loop.

## Cheat sheet

| Tool | Cost | Use when |
|---|---|---|
| `Start-Job` | new process per job | isolation matters more than speed |
| `Start-ThreadJob` | new thread, same process | many lightweight concurrent tasks |
| `ForEach-Object -Parallel` | new thread per item, simplest syntax | quick fan-out over a collection |
| `$using:variable` | required to read outer scope in any of the above | any variable a parallel block needs |
| `ConcurrentBag`/`ConcurrentDictionary`/`ConcurrentQueue` | thread-safe collections | accumulating results from parallel work |
| `New-PSSession` + `Invoke-Command -Session` | persistent remote connection | multiple related remote calls |
| `Invoke-Command -ComputerName @(...)` | parallel fan-out, per-machine | one-off command across many machines |
| `-ThrottleLimit` | caps concurrency | always set deliberately, don't rely on the default |

## Exercise

Write a script that takes a list of URLs and, using `ForEach-Object
-Parallel` with `-ThrottleLimit 5`, checks each with `Invoke-WebRequest`
(wrapped in try/catch so one failure doesn't stop the rest), accumulating
results (URL, status code, response time) into a `ConcurrentBag`. Compare
its total runtime against a plain sequential `foreach` loop doing the
same checks, using `Measure-Command`, and confirm the parallel version is
faster for a list of at least 10 URLs.
