# 08 · Script Modules & Manifests

[Level 1](../level-1/09-modules-basics.md) covered a bare `.psm1` file as a
module. Once a module is meant to be shared, versioned, or depended on by
other scripts, it needs a **manifest** — a `.psd1` file that describes the
module's metadata: version, author, required PowerShell version, and
exactly which functions it exposes. This module covers building a module
properly, the way real PowerShell Gallery packages are structured.

## Why a manifest matters

A bare `.psm1` works, but it can't answer basic questions without a human
reading its source: What version is this? What PowerShell version does it
need? What does it depend on? A manifest makes all of that queryable by
tooling — `Install-Module`, `Get-Module`, and CI pipelines all read it.

## The standard module folder layout

```text
LogUtils/
  LogUtils.psd1      # the manifest — metadata + which file to load
  LogUtils.psm1       # the actual code
```

The folder name, the `.psd1` name, and the `.psm1` name should all match —
this convention is what lets `Import-Module LogUtils` find everything
automatically once the folder is on `$env:PSModulePath`.

## Writing the module code

```powershell
# LogUtils/LogUtils.psm1
function Get-ErrorLines {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Path
    )
    Get-Content -Path $Path | Where-Object { $_ -match "ERROR" }
}

function Get-LineCount {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Path
    )
    (Get-Content -Path $Path).Count
}

Export-ModuleMember -Function Get-ErrorLines, Get-LineCount
```

## Generating a manifest with `New-ModuleManifest`

You could hand-write a `.psd1`, but `New-ModuleManifest` generates a
correctly-formatted one with sensible defaults for everything you don't
specify.

```powershell
New-ModuleManifest -Path "./LogUtils/LogUtils.psd1" `
    -RootModule "LogUtils.psm1" `
    -ModuleVersion "1.0.0" `
    -Author "Your Name" `
    -Description "Utilities for working with log files" `
    -FunctionsToExport @("Get-ErrorLines", "Get-LineCount") `
    -PowerShellVersion "7.0"
```

The resulting `LogUtils.psd1` is just a PowerShell hashtable saved to disk:

```powershell
@{
    RootModule        = 'LogUtils.psm1'
    ModuleVersion      = '1.0.0'
    GUID               = 'b2f1...'          # auto-generated, uniquely identifies the module
    Author             = 'Your Name'
    Description        = 'Utilities for working with log files'
    PowerShellVersion  = '7.0'
    FunctionsToExport  = @('Get-ErrorLines', 'Get-LineCount')
    CmdletsToExport    = @()
    VariablesToExport  = @()
    AliasesToExport    = @()
}
```

## Validating a manifest

```powershell
Test-ModuleManifest -Path "./LogUtils/LogUtils.psd1"
```

```text
ModuleType Version PreRelease Name     ExportedCommands
---------- ------- ---------- ----     ----------------
Script     1.0.0              LogUtils {Get-ErrorLines, Get-LineCount}
```

`Test-ModuleManifest` catches structural problems — a missing `RootModule`
file, an invalid version string, a `GUID` that isn't a valid GUID — before
you ever try to publish or share the module. Run it as part of any build or
CI step for a module.

## Importing by manifest vs by folder vs by `.psm1`

```powershell
Import-Module "./LogUtils/LogUtils.psd1"   # explicit: read the manifest
Import-Module "./LogUtils"                  # folder form: PowerShell finds the .psd1 itself
Import-Module "./LogUtils/LogUtils.psm1"    # bypasses the manifest entirely
```

Importing the `.psm1` directly still works, but it skips every check the
manifest would have enforced (minimum PowerShell version, required
modules) — always import by folder or manifest path once a manifest
exists, not the raw `.psm1`.

## `FunctionsToExport`: the trap of leaving it as `'*'`

```powershell
# Bad: exports EVERY function in the module, including internal helpers
FunctionsToExport = '*'

# Good: an explicit list — internal helper functions stay private
FunctionsToExport = @('Get-ErrorLines', 'Get-LineCount')
```

`New-ModuleManifest` defaults to `'*'` if you don't override it, which
silently exposes every function in the `.psm1` — including any private
helper functions you only meant for internal use within the module. Listing
exports explicitly (matching what `Export-ModuleMember` already says)
keeps the module's public surface intentional and small.

## Requiring dependencies

```powershell
@{
    RootModule        = 'DeployTools.psm1'
    ModuleVersion      = '2.1.0'
    RequiredModules    = @(
        @{ ModuleName = "Pester"; ModuleVersion = "5.0.0" }
    )
    PowerShellVersion  = '7.2'
}
```

`RequiredModules` makes `Import-Module` fail fast with a clear error if a
dependency isn't installed, rather than the module loading successfully but
failing mysteriously the first time it calls a function that doesn't
actually exist yet.

## Versioning: semantic version bumps matter

| Change | Version bump | Example |
|---|---|---|
| Bug fix, no behavior change to callers | Patch | `1.0.0` → `1.0.1` |
| New function/parameter added, old calls still work | Minor | `1.0.1` → `1.1.0` |
| Removed/renamed a function, or changed existing behavior | Major | `1.1.0` → `2.0.0` |

Callers can pin `RequiredModules` to a minimum version — bumping the major
version on a breaking change is how you signal "don't blindly auto-update
past this point" to anyone depending on the module.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `.psm1` | the module's actual code |
| `.psd1` (manifest) | metadata: version, author, exports, dependencies |
| `New-ModuleManifest` | generate a correctly-formatted `.psd1` |
| `Test-ModuleManifest` | validate a manifest before publishing/sharing |
| `FunctionsToExport` | explicit public function list — avoid leaving it as `'*'` |
| `RequiredModules` | fail fast if a dependency is missing |
| `Import-Module <folder>` | loads via the manifest automatically |

## Exercise

Turn the `MathHelpers.psm1` module from Level 1's module exercise (or write
a new one with `Get-Square` and `Get-Cube`) into a proper module folder:
`MathHelpers/MathHelpers.psm1` plus a generated `MathHelpers.psd1` with
`ModuleVersion "1.0.0"`, an explicit `FunctionsToExport` list, and
`PowerShellVersion "7.0"`. Run `Test-ModuleManifest` against it to confirm
it's valid, then `Import-Module` the folder and call both functions.
