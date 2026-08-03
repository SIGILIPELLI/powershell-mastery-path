# 05 · Remoting Basics

Everything so far has run commands on the machine you're sitting at.
PowerShell Remoting lets you run those same commands **on another machine**
— one target, or a hundred — and get real objects back, not just text
scraped from an SSH session. This module covers the core cmdlets:
`Invoke-Command` for one-off remote execution, and `PSSession` for a
persistent remote connection you can reuse.

## How remoting transports work

Classic Windows PowerShell remoting runs over **WinRM** (WS-Management, a
SOAP-based protocol over HTTP/HTTPS, port 5985/5986). PowerShell 7+ added
**SSH-based remoting**, which works the same way from the scripting side
but tunnels over an ordinary SSH connection — this is what makes remoting
practical from macOS/Linux to other macOS/Linux/Windows machines, since SSH
is already standard on non-Windows systems and WinRM generally isn't.

```powershell
# WinRM-based (classic, Windows-to-Windows)
Invoke-Command -ComputerName "SERVER01" -ScriptBlock { Get-Service }

# SSH-based (cross-platform, PowerShell 7+)
Invoke-Command -HostName "192.168.1.50" -UserName "deploy" -ScriptBlock { Get-Process }
```

Both produce identical results from the caller's point of view — the
difference is entirely in the transport and how the target machine is
configured to accept connections (`Enable-PSRemoting` for WinRM,
`sshd` + `PowerShell` installed on the remote host for SSH).

## `Invoke-Command`: run once, get objects back

```powershell
Invoke-Command -ComputerName "SERVER01", "SERVER02" -ScriptBlock {
    Get-Process | Where-Object { $_.WorkingSet64 -gt 200MB }
}
```

```text
ProcessName    Id  WorkingSet64 PSComputerName
-----------    --  ------------ --------------
chrome       4021     215466752 SERVER01
sqlservr     8842     892034000 SERVER02
```

Notice the extra `PSComputerName` property — remoting automatically tags
every returned object with which machine it came from, since a single call
can target many computers at once and the results are all merged into one
collection.

## Passing local variables into the remote script block

A remote script block runs in its own process on the target machine — it
has **no access** to your local session's variables unless you explicitly
pass them in with `-ArgumentList` (positional) or `$using:`.

```powershell
$thresholdMB = 200

# $using: reaches into the LOCAL session from inside a remote script block
Invoke-Command -ComputerName "SERVER01" -ScriptBlock {
    Get-Process | Where-Object { $_.WorkingSet64 -gt ($using:thresholdMB * 1MB) }
}
```

```powershell
# -ArgumentList maps positionally onto param() inside the script block
Invoke-Command -ComputerName "SERVER01" -ScriptBlock {
    param($MinMB)
    Get-Process | Where-Object { $_.WorkingSet64 -gt ($MinMB * 1MB) }
} -ArgumentList 200
```

Forgetting `$using:` is one of the most common remoting mistakes — a plain
`$thresholdMB` inside the remote script block would be `$null` there (it
only exists locally), and the comparison would silently misbehave instead
of throwing an obvious error.

## `PSSession`: a persistent connection you can reuse

`Invoke-Command` without a session opens a brand-new connection, runs the
script block, and tears the connection down — fine for one-off commands,
wasteful if you're running many commands against the same machine, and
useless if you need state (like a changed directory or an imported module)
to persist between calls.

```powershell
$session = New-PSSession -ComputerName "SERVER01"

# Reuse the same connection for multiple calls — state persists between them
Invoke-Command -Session $session -ScriptBlock { Set-Location "C:\Logs" }
Invoke-Command -Session $session -ScriptBlock { Get-Location }   # still C:\Logs

Remove-PSSession $session   # always clean up when done
```

```text
Path
----
C:\Logs
```

Without a session, each `Invoke-Command` call is a fresh, stateless
connection — `Set-Location` in one call would have no effect on a
following, separate `Invoke-Command` call, because there'd be no shared
process between them.

## `Enter-PSSession`: an interactive remote shell

```powershell
Enter-PSSession -ComputerName "SERVER01"
# prompt changes to [SERVER01]: PS C:\>
# now every command you type runs on SERVER01, interactively
Exit-PSSession
# prompt returns to normal
```

`Enter-PSSession` is the interactive equivalent of SSH-ing into a machine —
useful for exploring or troubleshooting by hand. `Invoke-Command` is what
you use inside actual scripts, since it's non-interactive and returns
structured objects your script can keep processing.

## Fan-out: running against many machines at once

```powershell
$servers = "SERVER01", "SERVER02", "SERVER03"

$results = Invoke-Command -ComputerName $servers -ScriptBlock {
    [pscustomobject]@{
        Hostname = $env:COMPUTERNAME
        Uptime   = (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
    }
}

$results | Select-Object PSComputerName, Uptime | Format-Table -AutoSize
```

By default, `Invoke-Command` runs against multiple computers **in
parallel**, not one after another — a health check against 50 servers takes
roughly as long as against 1, not 50 times as long. `-ThrottleLimit`
controls the maximum number of simultaneous connections if you need to cap
that.

## The trap: return values must be serializable

```powershell
Invoke-Command -ComputerName "SERVER01" -ScriptBlock {
    Get-Process -Name "notepad"
}
```

Objects returned from a remote session are **deserialized** copies — plain
data, not live objects. A live `System.Diagnostics.Process` object has a
`.Kill()` method; the deserialized copy that comes back over remoting does
not, because "kill this process" only makes sense on the machine the
process is actually running on. If you need to act on a remote process,
the action itself (like `Stop-Process`) has to happen **inside** the
remote script block, not after the result comes back locally.

```powershell
# Wrong: .Kill() doesn't exist on the deserialized object returned locally
$proc = Invoke-Command -ComputerName "SERVER01" -ScriptBlock { Get-Process notepad }
$proc.Kill()   # throws — method not available on a deserialized object

# Right: do the action on the remote machine, inside the script block
Invoke-Command -ComputerName "SERVER01" -ScriptBlock {
    Get-Process notepad | Stop-Process
}
```

## Copying files to/from a remote session

```powershell
Copy-Item -Path "C:\local\deploy.zip" -Destination "C:\remote\" -ToSession $session
Copy-Item -Path "C:\remote\report.json" -Destination "C:\local\" -FromSession $session
```

`Copy-Item` understands `-ToSession`/`-FromSession` against an existing
`PSSession`, so file transfer piggybacks on the same remoting
infrastructure instead of needing a separate protocol like SCP or SMB.

## Cheat sheet

| Cmdlet/Concept | Purpose |
|---|---|
| `Invoke-Command -ComputerName ...` | run a script block on one or more remote machines, one-shot |
| `New-PSSession` / `Remove-PSSession` | open/close a persistent remote connection |
| `Invoke-Command -Session $s` | reuse a persistent session, state carries over |
| `Enter-PSSession` / `Exit-PSSession` | interactive remote shell |
| `$using:variable` | pass a local variable into a remote script block |
| `-ArgumentList` | pass values positionally into a script block's `param()` |
| `PSComputerName` | auto-added property showing which machine a result came from |
| `-ThrottleLimit` | cap how many remote connections run in parallel |
| `Copy-Item -ToSession` / `-FromSession` | transfer files over an existing session |

## Exercise

Assuming you have two machines reachable over remoting (or use
`-ComputerName localhost` if remoting is enabled locally for testing),
write a script that opens a `PSSession` to each of a list of computer names,
runs a script block that collects `[pscustomobject]` with `Hostname`,
`OSVersion` (`$PSVersionTable.OS`), and `FreeMemoryMB`, then closes every
session with `Remove-PSSession`. Print the combined results as a table
sorted by `FreeMemoryMB` ascending.
