# 09 · Modules Basics

A **module** is a packaged, reusable unit of PowerShell code — functions,
variables, and more — that you can load with `Import-Module`. Every cmdlet
you've used so far (`Get-Process`, `Get-Content`, ...) actually lives in a
built-in module.

## Seeing what's loaded and available

```powershell
Get-Module                          # modules currently loaded in this session
Get-Module -ListAvailable            # all modules installed on this machine
```

```powershell
# find which module a cmdlet comes from
Get-Command Get-Process | Select-Object Name, ModuleName
```

```text
Name        ModuleName
----        ----------
Get-Process Microsoft.PowerShell.Management
```

## Installing modules from the PowerShell Gallery

The PowerShell Gallery is the community package repository (like PyPI for
Python or npm for JavaScript).

```powershell
# search for a module
Find-Module -Name "Pester"

# install for the current user (no admin rights needed)
Install-Module -Name Pester -Scope CurrentUser -Force

# import it into the current session
Import-Module Pester
```

## Writing your own simple module

A module can be as simple as a `.psm1` file containing one or more
functions.

```powershell
# MathHelpers.psm1
function Get-Square {
    param([int]$Number)
    return $Number * $Number
}

function Get-Cube {
    param([int]$Number)
    return $Number * $Number * $Number
}

Export-ModuleMember -Function Get-Square, Get-Cube
```

```powershell
# using it from another script in the same folder
Import-Module ./MathHelpers.psm1

Get-Square -Number 5    # 25
Get-Cube -Number 3       # 27
```

`Export-ModuleMember` controls what's visible from outside the module —
without it, every function defined in the file is exported by default, but
being explicit is good practice once a module grows.

## Module search paths

```powershell
$env:PSModulePath -split [System.IO.Path]::PathSeparator
```

Modules placed in one of these standard locations (a per-user modules
folder, in a subfolder matching the module's own name) can be imported by
name alone, without a path:

```powershell
Import-Module MathHelpers    # works if MathHelpers/MathHelpers.psm1 is on $env:PSModulePath
```

## Removing a module from the session

```powershell
Remove-Module MathHelpers
```

This unloads the module's functions from the current session (useful while
iterating on a module's code) without uninstalling anything from disk.

## Dot-sourcing vs importing

For a *single script file* full of helper functions (not a full module),
you can also "dot-source" it — this runs the file in the *current* scope
rather than a separate module scope.

```powershell
# helpers.ps1
function Get-Greeting { param($Name) "Hello, $Name!" }
```

```powershell
# main.ps1
. ./helpers.ps1        # note the leading ". " with a space

Get-Greeting -Name "Ada"   # Hello, Ada!
```

Dot-sourcing is simple and fine for small scripts; real reusable code that
you'll `Import-Module` from many places should be a proper module (covered
in depth in Level 3).

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `Get-Module` / `Get-Module -ListAvailable` | list loaded / installed modules |
| `Find-Module` | search the PowerShell Gallery |
| `Install-Module -Scope CurrentUser` | install a module for your user |
| `Import-Module <name-or-path>` | load a module into the session |
| `Export-ModuleMember -Function ...` | control what a module exposes |
| `Remove-Module` | unload a module from the session |
| `. ./script.ps1` | dot-source a script into the current scope |

## Exercise

Create a `StringHelpers.psm1` module with two functions —
`ConvertTo-TitleCaseWords` (splits a sentence into words and title-cases
each) and `Get-WordCount` (returns the number of words in a string) — export
both with `Export-ModuleMember`, then write a separate `test.ps1` that
imports the module and calls both functions on a sample sentence.
