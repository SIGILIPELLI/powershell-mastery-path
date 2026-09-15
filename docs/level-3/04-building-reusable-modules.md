---
description: "Building Reusable Modules — Level 2 covered a single .psm1 with a manifest. Real modules grow past one file fast — a dozen public functions, a handful of…"
---

# 04 · Building Reusable Modules

Level 2 covered a single `.psm1` with a manifest. Real modules grow past
one file fast — a dozen public functions, a handful of private helpers
they share, and a clear line between "what callers can use" and "internal
plumbing that might change." This module builds that structure properly.

## Layout: one function per file, split by visibility

```text
StringToolkit/
├── StringToolkit.psd1
├── StringToolkit.psm1
├── Public/
│   ├── ConvertTo-TitleCase.ps1
│   └── ConvertTo-SlugCase.ps1
└── Private/
    └── Remove-ExtraWhitespace.ps1
```

`Public/` holds functions meant to be called by consumers of the module;
`Private/` holds helpers those public functions share internally but that
nobody outside the module should depend on. One function per file keeps
diffs small and makes the module easy to navigate as it grows.

```powershell
# Private/Remove-ExtraWhitespace.ps1
function Remove-ExtraWhitespace {
    param([string]$Text)
    ($Text -replace '\s+', ' ').Trim()
}
```

```powershell
# Public/ConvertTo-TitleCase.ps1
function ConvertTo-TitleCase {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string]$Text
    )
    process {
        $clean = Remove-ExtraWhitespace -Text $Text
        (Get-Culture).TextInfo.ToTitleCase($clean.ToLower())
    }
}
```

```powershell
# Public/ConvertTo-SlugCase.ps1
function ConvertTo-SlugCase {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string]$Text
    )
    process {
        $clean = Remove-ExtraWhitespace -Text $Text
        ($clean.ToLower() -replace '[^a-z0-9]+', '-').Trim('-')
    }
}
```

A public function calling a private one is completely normal — private
functions aren't isolated, they're just not exported.

## The loader: `.psm1` becomes dot-sourcing glue

```powershell
# StringToolkit.psm1
$here = $PSScriptRoot
foreach ($folder in 'Private', 'Public') {
    $path = Join-Path $here $folder
    if (Test-Path $path) {
        Get-ChildItem -Path $path -Filter '*.ps1' | ForEach-Object {
            . $_.FullName
        }
    }
}

$publicFunctions = Get-ChildItem -Path (Join-Path $here 'Public') -Filter '*.ps1' |
    ForEach-Object { $_.BaseName }

Export-ModuleMember -Function $publicFunctions
```

Loading `Private` before `Public` matters: public functions call private
ones, so the private definitions need to exist first. `Export-ModuleMember`
is what actually enforces the boundary — every file gets dot-sourced and
defined either way, but only names passed to `-Function` become visible
outside the module.

## The manifest, pointing `FunctionsToExport` at the real list

```powershell
# StringToolkit.psd1
@{
    RootModule        = 'StringToolkit.psm1'
    ModuleVersion     = '1.0.0'
    GUID              = 'b3f1a2e0-1111-4a2b-9c3d-abcdef012345'
    Author            = 'Mastery Path'
    Description       = 'Small reusable string-formatting helpers.'
    PowerShellVersion = '5.1'
    FunctionsToExport = @('ConvertTo-TitleCase', 'ConvertTo-SlugCase')
    CmdletsToExport   = @()
    VariablesToExport = @()
    AliasesToExport   = @()
}
```

`FunctionsToExport` here is a second, independent gate on top of
`Export-ModuleMember` — for a published module, list names explicitly
(rather than `'*'`) so `Import-Module` can resolve exports without
loading the whole module first, which speeds up tab-completion and
`Get-Command` discovery.

## Importing and verifying the boundary

```powershell
Import-Module ./StringToolkit -Force

'  the QUICK brown   fox ' | ConvertTo-TitleCase
'  the QUICK brown   fox! ' | ConvertTo-SlugCase
```

```text
The Quick Brown Fox
the-quick-brown-fox
```

```powershell
Get-Command -Module StringToolkit
```

```text
CommandType     Name                    Version Source
-----------     ----                    ------- ------
Function        ConvertTo-SlugCase      1.0.0   StringToolkit
Function        ConvertTo-TitleCase     1.0.0   StringToolkit
```

Only the two public functions show up — `Remove-ExtraWhitespace` was
defined (dot-sourced) but never exported:

```powershell
try { Remove-ExtraWhitespace -Text 'x' }
catch { "Private not exported: $($_.Exception.Message)" }
```

```text
Private not exported: The term 'Remove-ExtraWhitespace' is not recognized
as a name of a cmdlet, function, script file, or executable program.
```

## The trap: dot-sourcing pollutes the caller's scope, not just the module's

```powershell
# Wrong: this defines functions in the CALLING script's scope, bypassing the module system entirely
. ./StringToolkit/Public/ConvertTo-TitleCase.ps1
```

Dot-sourcing a file directly (outside the `.psm1` loader) runs it in
whatever scope you dot-sourced from — if that's your interactive session
or another script, the function is now defined there permanently, with no
`Export-ModuleMember` boundary, no version, and no way to `Remove-Module`
it later. Always `Import-Module` the module folder; only the module's own
`.psm1` should dot-source its component files.

## Versioning and `Remove-Module`/`Import-Module -Force`

```powershell
Get-Module StringToolkit | Remove-Module
Import-Module ./StringToolkit -Force
```

`-Force` re-imports even if a module of the same name is already loaded —
essential while developing, since PowerShell otherwise silently keeps
serving the *old* in-memory version after you edit a source file.
`Remove-Module` first guarantees a truly clean reload, useful when you've
renamed or deleted a function and want to confirm it's actually gone.

## Cheat sheet

| Piece | Role |
|---|---|
| `Public/*.ps1` | one function per file, exported to callers |
| `Private/*.ps1` | shared internal helpers, never exported |
| `.psm1` | loader: dot-sources every file, then `Export-ModuleMember` |
| `.psd1` `FunctionsToExport` | second export gate, also speeds up discovery |
| `Export-ModuleMember -Function` | the actual visibility boundary |
| `Import-Module -Force` | reload after edits (module caching otherwise hides changes) |
| `Get-Command -Module Name` | confirm exactly what's exported |
| Dot-sourcing outside `.psm1` | anti-pattern — bypasses the whole module boundary |

## How It Actually Works

A multi-file module (a `.psm1` that dot-sources or `Import`s several
`Private`/`Public` `.ps1` files) still ends up as **one flat child session
state** — every function defined by every dot-sourced file lands in the
same module-scoped function table, which is why the common "loop over
`Public/*.ps1` and `Private/*.ps1`, dot-source each" pattern works: dot-
sourcing inside the module's own script runs each file's code directly in
the module's session state (not a further nested child state), so
functions defined in `Private/Helper.ps1` are immediately visible to code
in `Public/DoThing.ps1`, exactly as if they'd all been typed into one
`.psm1` file — the file split is purely an authoring convenience with no
runtime scoping consequence.

What actually enforces "public vs. private" is `Export-ModuleMember`
(or the manifest's `FunctionsToExport`), which controls what the module's
**exported command table** — a separate list the engine attaches to the
`PSModuleInfo` object — exposes to the *importing* session's global scope.
Everything not listed there stays reachable from other functions *inside*
the module (since they share session state) but is absent from the
caller's command table entirely, which is why calling an unexported
helper from outside the module fails with `CommandNotFoundException`
rather than an access-denied error — as far as the caller's scope is
concerned, the command simply doesn't exist.

`Remove-Module` followed by `Import-Module -Force` (rather than just
`-Force` alone) matters because `-Force` re-runs the module script but
doesn't necessarily start from a completely clean session state if
stale script-scoped variables or event subscriptions from the previous
load are still referenced elsewhere (e.g., registered `Register-
ObjectEvent` handlers) — `Remove-Module` explicitly runs the module's
`OnRemove` script block and tears down its session state object before
the next import builds a fresh one, which is the more reliable way to get
a truly clean reload during iterative development.

## Exercise

Build a `MathToolkit` module with the same Public/Private split: a public
`Get-Statistics` function that returns min/max/mean/median for an array of
numbers, backed by a private `Get-Median` helper. Write the manifest with
an explicit `FunctionsToExport`, import it with `-Force`, confirm
`Get-Command -Module MathToolkit` only lists the public function, and
confirm calling `Get-Median` directly from outside the module fails.
