# 08 · Publishing Modules (PowerShell Gallery)

A module that only ever lives in one repo's `Import-Module -Force` cycle
hasn't really shipped. Publishing to the PowerShell Gallery — the
`Install-Module`/`Find-Module` ecosystem every PowerShell user already
has access to — means anyone can `Install-Module YourModule` the same way
they'd install `Pester` or `Az`. This module covers getting a module
ready for that, and what actually happens on publish (without needing a
real Gallery API key to follow along).

## A complete manifest, generated properly

```powershell
New-ModuleManifest -Path './GreetingKit/GreetingKit.psd1' `
    -RootModule 'GreetingKit.psm1' `
    -ModuleVersion '1.0.0' `
    -Author 'Mastery Path' `
    -Description 'A tiny demo module for the PowerShell Gallery publishing lesson.' `
    -FunctionsToExport 'Get-Greeting' `
    -Tags 'demo', 'greeting' `
    -ProjectUri 'https://example.com/greetingkit' `
    -LicenseUri 'https://example.com/license'
```

`New-ModuleManifest` is worth using over hand-writing a `.psd1` from
scratch — it fills in every field the Gallery actually reads
(`Tags`, `ProjectUri`, `LicenseUri`, `ReleaseNotes`) with correct syntax,
and Gallery search/discoverability depends on those being present and
accurate. `Tags` in particular is how `Find-Module -Tag 'demo'` finds your
module at all.

```powershell
Test-ModuleManifest -Path './GreetingKit/GreetingKit.psd1'
```

```text
ModuleType Version   PreRelease Name          ExportedCommands
---------- -------   ---------- ----          ----------------
Script     1.0.0                GreetingKit   Get-Greeting
```

`Test-ModuleManifest` parses the file the same way `Import-Module` and
`Publish-Module` will — running it before publishing catches a malformed
manifest immediately instead of discovering the problem after a failed
(or worse, partially succeeded) publish.

## Linting before publishing: `PSScriptAnalyzer`

```powershell
Invoke-ScriptAnalyzer -Path ./GreetingKit -Recurse
```

```text
RuleName                     Severity    ScriptName       Line Message
--------                     --------    ----------       ---- -------
PSUseToExportFieldsInManifest Warning    GreetingKit.psd1  75  Do not use wildcard
                                                                or $null in this
                                                                field. Explicitly
                                                                specify a list for
                                                                CmdletsToExport.
PSUseToExportFieldsInManifest Warning    GreetingKit.psd1  81  ...for AliasesToExport.
PSProvideCommentHelp        Information GreetingKit.psm1    1  The cmdlet
                                                                'Get-Greeting' does
                                                                not have a help
                                                                comment.
```

Real findings against a genuinely minimal module: `New-ModuleManifest`'s
own defaults leave `CmdletsToExport`/`AliasesToExport` as `'*'`, which
`PSScriptAnalyzer` correctly flags — an explicit empty array (`@()`) or a
real list is faster for `Import-Module` to resolve and clearer about
intent. The `PSProvideCommentHelp` finding is a nudge, not an error:
adding `<# .SYNOPSIS ... #>` comment-based help (module 01, Level 3)
means `Get-Help Get-Greeting` works for anyone who installs it, which
matters far more once "anyone" is a real Gallery audience rather than
just your own team. Run `Invoke-ScriptAnalyzer -Recurse` as a required
step before every publish — most published-module quality problems are
things this catches automatically.

## What `Publish-Module` actually does

```powershell
Publish-Module -Path ./GreetingKit -NuGetApiKey $env:GALLERY_API_KEY -Verbose
```

This isn't run here — it needs a real PowerShell Gallery account and API
key, and would genuinely publish a package to a public registry, which
this course won't do without you deciding to. What it does, so the step
isn't a mystery: it packages the module folder as a NuGet package,
uploads it using your API key (obtained from your Gallery account
settings, never hardcoded — the same `SecureString`/environment-variable
discipline from Level 3's security module applies directly here), and the
version in `.psd1`'s `ModuleVersion` becomes the published version. **A
version number can never be republished or overwritten** — every publish
needs to bump `ModuleVersion` first, even for a one-line fix.

## Versioning: semantic versioning matters here specifically

```powershell
# .psd1
ModuleVersion = '1.2.3'
```

Format is `Major.Minor.Patch`. Increment:
- **Patch** (`1.2.3` → `1.2.4`) for a bug fix, no behavior change to
  callers
- **Minor** (`1.2.3` → `1.3.0`) for a new function/parameter that doesn't
  break existing callers
- **Major** (`1.2.3` → `2.0.0`) for anything that could break someone
  already depending on the current behavior — a renamed parameter,
  changed return type, removed function

Anyone who ran `Install-Module YourModule` is trusting that a minor/patch
bump won't break their script; violating that (a breaking change without
a major version bump) is the fastest way to lose a Gallery module's
users' trust.

## `#Requires` and dependency declarations

```powershell
#Requires -Version 7.0
#Requires -Modules @{ ModuleName = 'PSFramework'; ModuleVersion = '1.7.0' }
```

At the top of a script or module file, `#Requires` makes PowerShell
refuse to even attempt to run the file if the version/dependency isn't
met, with a clear error naming exactly what's missing — far better than
letting the script run partway and fail confusingly on the first call
that needed the missing piece. For a module with real dependencies,
declare them in the manifest's `RequiredModules` too, so
`Install-Module YourModule` pulls them in automatically.

```powershell
# in the .psd1
RequiredModules = @(
    @{ ModuleName = 'PSFramework'; ModuleVersion = '1.7.0' }
)
```

## The trap: publishing with local/test artifacts still in the folder

A module folder that also contains `./Tests/`, a `./bin/` build output
directory, or a stray `.vscode/` config gets packaged and published
alongside the actual module — harmless usually, but it bloats the
package and can occasionally leak something not meant to ship (a local
test fixture with fake-but-realistic-looking credentials, for instance).
`Publish-Module -Path` publishes *everything* under that path; keep the
publish path scoped to exactly the module's real content, or use a
`.gitignore`-style exclude/staging step that copies only what should ship
into a clean folder before calling `Publish-Module` on that.

## Cheat sheet

| Step | Tool |
|---|---|
| Generate a correct manifest | `New-ModuleManifest` |
| Validate it parses correctly | `Test-ModuleManifest` |
| Lint before publishing | `Invoke-ScriptAnalyzer -Recurse` |
| Publish | `Publish-Module -Path ... -NuGetApiKey ...` |
| Bump version before every publish | `ModuleVersion` in `.psd1`, semantic versioning |
| Declare PowerShell/module dependencies | `#Requires`, `RequiredModules` in manifest |
| Keep the publish folder clean | scope `-Path` to only real module content |

## Exercise

Take the `AdminToolkit` module from Level 3's project. Generate its
manifest properly with `New-ModuleManifest` (rather than the hand-written
one from that lesson), run `Invoke-ScriptAnalyzer -Recurse` against it and
fix every `Warning`-or-higher finding, add comment-based help to each
public function to resolve the `PSProvideCommentHelp` findings, and bump
its `ModuleVersion` to `1.1.0` to reflect the new documentation as a
minor, non-breaking addition.
