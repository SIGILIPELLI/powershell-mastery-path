# 01 · Setup & First Script

PowerShell today means **PowerShell 7+** (also called "PowerShell Core"),
built on .NET and run with the `pwsh` executable. It's open source and
cross-platform — the same scripts run on Windows, macOS, and Linux. This is
different from **Windows PowerShell 5.1**, the older, Windows-only, built-in
version (`powershell.exe`). Throughout this course, "PowerShell" means
PowerShell 7+ unless noted otherwise.

## Installing PowerShell 7+

```powershell
# Windows (winget)
winget install --id Microsoft.PowerShell --source winget

# macOS (Homebrew)
brew install --cask powershell

# Linux (Ubuntu/Debian) — see Microsoft's docs for your distro's exact steps
sudo apt-get install -y powershell
```

Verify the install:

```powershell
pwsh --version
# PowerShell 7.4.1
```

## Starting a session

Launch `pwsh` from any terminal to get an interactive session — a **REPL**
(Read-Eval-Print Loop) where you can type commands one at a time and see
results immediately.

```powershell
pwsh
# PowerShell 7.4.1
# PS /Users/you>
```

```powershell
# try a built-in cmdlet right away
Get-Date
```

```text
# Expected output (abbreviated)
Saturday, July 20, 2026 9:12:03 AM
```

## Your first script file

Scripts live in `.ps1` files. Create one with any text editor:

```powershell
# hello.ps1
Write-Host "Hello, PowerShell!"
Write-Output "This line is returned as output, not just printed."
```

Run it:

```powershell
pwsh ./hello.ps1
# or, from inside an already-running pwsh session:
./hello.ps1
```

```text
Hello, PowerShell!
This line is returned as output, not just printed.
```

`Write-Host` writes directly to the console (for humans) and cannot be
captured or piped onward. `Write-Output` sends an object down the pipeline —
it can be captured in a variable, piped to another cmdlet, or redirected to a
file. Prefer `Write-Output` (or simply letting an expression's result fall
through) for anything that might be consumed by other code; reserve
`Write-Host` for pure console messages like progress notes.

## Execution policy

On Windows, PowerShell blocks running local scripts by default as a safety
measure. Check and adjust it with:

```powershell
Get-ExecutionPolicy
# Restricted

# Allow locally-created scripts to run for the current user only
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

`RemoteSigned` is the common safe default: local scripts run freely, but
scripts downloaded from the internet must be digitally signed. macOS and
Linux builds of `pwsh` do not enforce this policy the same way, so you'll
mostly encounter it on Windows.

## Comments and basic syntax

```powershell
# a single-line comment

<#
  a block comment,
  spanning multiple lines
#>

Write-Output "PowerShell statements don't need a trailing semicolon,"
Write-Output "but you can use one to put two statements on one line."; Write-Output "like this"
```

## The help system

PowerShell's built-in help is one of its best features — every cmdlet is
self-documenting.

```powershell
Get-Help Get-Process
Get-Help Get-Process -Examples
Get-Help Get-Process -Full

# first time only, to download the latest help content
Update-Help
```

```powershell
# discover cmdlets by keyword
Get-Command -Noun Process
```

```text
CommandType     Name                    Version    Source
-----------     ----                    -------    ------
Cmdlet          Get-Process             7.0.0.0    Microsoft.PowerShell.Management
Cmdlet          Stop-Process            7.0.0.0    Microsoft.PowerShell.Management
...
```

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `pwsh` | start an interactive PowerShell 7+ session |
| `pwsh ./script.ps1` | run a script from the shell |
| `Get-ExecutionPolicy` / `Set-ExecutionPolicy` | check / change script-running permissions |
| `Write-Output` | send an object down the pipeline |
| `Write-Host` | print directly to the console (not pipeable) |
| `Get-Help <cmdlet>` | show documentation for a cmdlet |
| `Get-Command -Noun <thing>` | find cmdlets related to a noun |

## Exercise

Write `greet.ps1` that uses `Write-Output` to print a greeting, then calls
`Get-Date` and stores the result in a variable, then prints a second message
that includes that variable so it reads something like
`"Generated on <date>."`. Run it with `pwsh ./greet.ps1` and confirm both
lines appear.
