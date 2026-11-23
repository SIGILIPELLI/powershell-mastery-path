---
title: "PowerShell Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free PowerShell cheat sheet: 37 quick-reference tables from every level of the PowerShell Mastery Path, each linked to its full lesson."
---

# PowerShell Cheat Sheet

Quick-reference tables from **37 lessons** of the free PowerShell Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; First Script](level-1/01-setup.md)

| Command | Purpose |
|---------|---------|
| `pwsh` | start an interactive PowerShell 7+ session |
| `pwsh ./script.ps1` | run a script from the shell |
| `Get-ExecutionPolicy` / `Set-ExecutionPolicy` | check / change script-running permissions |
| `Write-Output` | send an object down the pipeline |
| `Write-Host` | print directly to the console (not pipeable) |
| `Get-Help <cmdlet>` | show documentation for a cmdlet |
| `Get-Command -Noun <thing>` | find cmdlets related to a noun |

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Type literal | .NET type | Example |
|--------------|-----------|---------|
| `[int]` | `System.Int32` | `[int]$x = 5` |
| `[double]` | `System.Double` | `[double]$pi = 3.14` |
| `[string]` | `System.String` | `[string]$s = "hi"` |
| `[bool]` | `System.Boolean` | `[bool]$flag = $true` |
| `[array]` | `System.Object[]` | `[array]$a = 1,2,3` |
| `[hashtable]` | `System.Collections.Hashtable` | `[hashtable]$h = @{}` |
| `[datetime]` | `System.DateTime` | `[datetime]$d = Get-Date` |

### [03 · Control Flow](level-1/03-control-flow.md)

| Operator | Meaning | Example |
|----------|---------|---------|
| `-eq` | equal | `$a -eq $b` |
| `-ne` | not equal | `$a -ne $b` |
| `-lt` / `-le` | less than / or equal | `$a -lt $b` |
| `-gt` / `-ge` | greater than / or equal | `$a -gt $b` |
| `-like` | wildcard match (string) | `$name -like "A*"` |
| `-match` | regex match (string) | `$name -match "^A"` |
| `-contains` | collection contains value | `$arr -contains 5` |
| `-in` | value is in collection | `5 -in $arr` |

All comparison operators are case-*insensitive* by default for strings; add
an `i`/`c` prefix to be explicit: `-ieq` (insensitive, default) vs `-ceq`
(case-sensitive).

```powershell
"Apple" -eq "apple"     # True  (case-insensitive by default)
"Apple" -ceq "apple"    # False (case-sensitive variant)
```

### [04 · Functions](level-1/04-functions.md)

| Syntax | Meaning |
|--------|---------|
| `function Verb-Noun { }` | define a function |
| `param($x, $y = 1)` | declare parameters with an optional default |
| `[Parameter(Mandatory)]` | require the caller to supply a value |
| `[ValidateRange(min,max)]` | validate a parameter's value |
| `return $value` | early exit with a value (optional otherwise) |
| `[CmdletBinding()]` | make a function behave like a real cmdlet |
| `process { }` | per-item block for pipeline input |
| `@params` | splat a hashtable as named arguments |

### [05 · Working with Objects &amp; the Pipeline](level-1/05-objects-pipeline.md)

| Cmdlet | Purpose |
|--------|---------|
| `Get-Member` | list an object's properties and methods |
| `Select-Object` | pick specific properties, or `-First`/`-Last` N objects |
| `Where-Object` | filter objects by a condition |
| `Sort-Object` | sort objects by a property |
| `Group-Object` | group objects by a shared property value |
| `ForEach-Object` | run a script block per pipeline object |
| `Measure-Object` | compute count/sum/average/min/max |
| `Format-Table` / `Format-List` | control display (not the underlying data) |
| `$_` | the current object in the pipeline |

### [06 · Arrays &amp; Hashtables](level-1/06-arrays-hashtables.md)

| Syntax | Meaning |
|--------|---------|
| `$a = 1,2,3` | create an array |
| `$a[0]` / `$a[-1]` | index / last element |
| `$a[1..3]` | slice |
| `$a.Count` | number of elements |
| `[System.Collections.Generic.List[T]]::new()` | growable, strongly-typed list |
| `@{ k = v }` | create a hashtable |
| `$h["key"]` / `$h.key` | access a value |
| `$h.ContainsKey("k")` | check key existence |
| `$h.GetEnumerator()` | iterate key/value pairs |
| `[ordered]@{}` | hashtable that preserves insertion order |

### [07 · Working with Files](level-1/07-working-with-files.md)

| Command | Purpose |
|---------|---------|
| `Test-Path` | check whether a file/directory exists |
| `Get-Content` | read a file (array of lines, or `-Raw` for one string) |
| `Set-Content` / `Add-Content` | overwrite / append text |
| `New-Item -ItemType Directory` | create a directory |
| `Get-ChildItem` | list files/directories (`ls`/`dir` alias) |
| `Copy-Item` / `Move-Item` / `Remove-Item` | copy / move / delete |
| `Get-Item` | get metadata about a file/directory |
| `Split-Path` / `Join-Path` | decompose / build file paths |

### [08 · Error Handling Basics](level-1/08-error-handling-basics.md)

| Concept | Meaning |
|---------|---------|
| Terminating error | stops the current scope; catchable with `try/catch` |
| Non-terminating error | prints and continues; needs `-ErrorAction Stop` to be catchable |
| `try { } catch { } finally { }` | structured error handling |
| `$_` / `$PSItem` in `catch` | the caught `ErrorRecord` |
| `throw "message"` | raise your own terminating error |
| `$Error` | session-wide list of past errors, most recent first |
| `-ErrorAction Stop/SilentlyContinue/Continue/Inquire` | control how non-terminating errors behave |

### [09 · Modules Basics](level-1/09-modules-basics.md)

| Command | Purpose |
|---------|---------|
| `Get-Module` / `Get-Module -ListAvailable` | list loaded / installed modules |
| `Find-Module` | search the PowerShell Gallery |
| `Install-Module -Scope CurrentUser` | install a module for your user |
| `Import-Module <name-or-path>` | load a module into the session |
| `Export-ModuleMember -Function ...` | control what a module exposes |
| `Remove-Module` | unload a module from the session |
| `. ./script.ps1` | dot-source a script into the current scope |

## Level 2 · Intermediate

### [01 · Advanced Pipeline](level-2/01-advanced-pipeline.md)

| Concept | What it means |
|---|---|
| Streaming pipeline | objects flow one at a time; memory stays flat |
| `Sort-Object` / `Group-Object` | must buffer everything first — no streaming |
| `$_` / `$PSItem` | current pipeline object; capture it before nesting pipelines |
| `begin { }` | runs once, before the first object |
| `process { }` | runs once **per** object |
| `end { }` | runs once, after the last object |
| `ValueFromPipeline` | binds the whole object to one parameter |
| `ValueFromPipelineByPropertyName` | binds a same-named property to a parameter |
| `return` inside `ForEach-Object` | exits only the current object's block call |

### [02 · Cmdlet Design](level-2/02-cmdlet-design.md)

| Concept | Purpose |
|---|---|
| `Get-Verb` | list PowerShell's approved verbs |
| `[CmdletBinding()]` | unlocks `-Verbose`/`-Debug`/`-ErrorAction`/etc. |
| `[ValidateNotNullOrEmpty()]`, `[ValidateRange]`, `[ValidateSet]`, `[ValidatePattern]`, `[ValidateScript]` | fail fast on bad input |
| `ParameterSetName` | mutually exclusive groups of parameters |
| `$PSCmdlet.ParameterSetName` | which set was actually used |
| `SupportsShouldProcess` + `$PSCmdlet.ShouldProcess(...)` | enables `-WhatIf`/`-Confirm` |
| `[OutputType(...)]` | documents the return type for tooling |

### [03 · Error Handling Advanced](level-2/03-error-handling-advanced.md)

| Concept | Behavior |
|---|---|
| Terminating error | stops the current scope; catchable with `try/catch` |
| Non-terminating error | reports and continues; needs `-ErrorAction Stop` to be catchable |
| `$ErrorActionPreference = "Stop"` | promotes all cmdlets' errors to terminating by default |
| `catch [SpecificType] { }` | order most-specific to least-specific, generic `catch` last |
| Custom exception class | lets callers `catch` your error type and read structured data |
| `throw "msg"` | raises a new terminating error |
| `throw` (bare, in `catch`) | re-raises the original error, preserving its type/stack |
| `Write-Error` | non-terminating; reports and continues |
| `trap { }` | older scope-wide error handler; prefer `try/catch` in new code |

### [04 · Regular Expressions in PowerShell](level-2/04-regular-expressions.md)

| Operator/Tool | Purpose |
|---|---|
| `-match` / `-notmatch` | test a string against a regex, populates `$Matches` |
| `-cmatch` | case-sensitive version of `-match` |
| `-replace` | regex-powered find-and-replace |
| `-creplace` | case-sensitive version of `-replace` |
| `$Matches['name']` | read a named capture group after a successful `-match` |
| `Select-String` | grep-like search across strings/files, returns rich objects |
| `[regex]::Matches($s, $p)` | get **every** match, not just the first |
| `[regex]::Escape($s)` | treat a string as a literal pattern, not regex syntax |

### [05 · Remoting Basics](level-2/05-remoting-basics.md)

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

### [06 · Testing with Pester](level-2/06-testing-pester.md)

| Concept | Purpose |
|---|---|
| `Describe` | groups all tests for one unit (usually a function) |
| `Context` | sub-groups related scenarios inside a `Describe` |
| `It` | a single test case, named in plain English |
| `Should -Be`, `-BeTrue`, `-Throw`, `-Contain`, `-Match` | common assertions |
| `BeforeAll` / `AfterAll` | run once per `Describe`, before/after all tests |
| `BeforeEach` / `AfterEach` | run before/after **every** `It` |
| `Mock` | replace a real command with a fake for the test |
| `Should -Invoke ... -Times` | verify a mocked command was called N times |
| `-ForEach @(...)` | run one `It` block once per data row |
| `Invoke-Pester -Path ... -Output Detailed` | run tests from the command line |

### [07 · Working with JSON/REST APIs](level-2/07-json-rest-apis.md)

| Cmdlet/Parameter | Purpose |
|---|---|
| `ConvertTo-Json -Depth N` | serialize an object to a JSON string; always set `-Depth` explicitly for nested data |
| `ConvertFrom-Json` | parse a JSON string into `[pscustomobject]` |
| `Invoke-RestMethod` | call an API, get the parsed JSON body back directly |
| `Invoke-WebRequest` | call an API, get the raw response (status, headers, raw content) |
| `-Method Post/Put/Delete` | HTTP verb to use |
| `-Body` | request payload (JSON-encode it first) |
| `-ContentType "application/json"` | tells the server the body's format |
| `-Headers @{...}` | custom headers, e.g. `Authorization` |
| `-ResponseHeadersVariable` / `-StatusCodeVariable` | capture headers/status alongside the parsed body |
| `-ErrorAction Stop` | required for `try/catch` to catch a non-2xx response |

### [08 · Script Modules &amp; Manifests](level-2/08-script-modules-manifests.md)

| Concept | Purpose |
|---|---|
| `.psm1` | the module's actual code |
| `.psd1` (manifest) | metadata: version, author, exports, dependencies |
| `New-ModuleManifest` | generate a correctly-formatted `.psd1` |
| `Test-ModuleManifest` | validate a manifest before publishing/sharing |
| `FunctionsToExport` | explicit public function list — avoid leaving it as `'*'` |
| `RequiredModules` | fail fast if a dependency is missing |
| `Import-Module <folder>` | loads via the manifest automatically |

### [09 · Working with CSV/Excel Data](level-2/09-csv-excel-data.md)

| Cmdlet/Parameter | Purpose |
|---|---|
| `Export-Csv -NoTypeInformation` | write objects to CSV (always include this switch) |
| `Import-Csv` | read CSV rows back as objects — **all properties are strings** |
| `-Delimiter ";"` | use a non-comma separator |
| `-Append` | add rows without rewriting the header |
| `[int]$_.Property` | cast before sorting/comparing numeric-looking CSV columns |
| `Measure-Object -Sum` | one of the few cmdlets that auto-coerces numeric strings |
| `Export-Excel` / `Import-Excel` (ImportExcel module) | real `.xlsx` files with proper types |

## Level 3 · Advanced

### [01 · Advanced Functions &amp; Parameter Validation](level-3/01-advanced-functions-validation.md)

| Feature | Purpose |
|---|---|
| `[CmdletBinding()]` | opt into common parameters, strict binding, pipeline blocks |
| `[Parameter(Mandatory)]` | require a value; prompts interactively if missing |
| `ValueFromPipeline` | bind the whole pipeline object to this parameter |
| `ValueFromPipelineByPropertyName` | bind a same-named property of the pipeline object |
| `ValidateSet`, `ValidateRange`, `ValidatePattern`, `ValidateScript` | reject bad values before the body runs |
| `begin` | runs once, before any pipeline item |
| `process` | runs once per pipeline item |
| `end` | runs once, after all pipeline items |
| `$_` / `$PSItem` | current item inside an active pipeline context only |
| Comment-based help (`<# .SYNOPSIS ... #>`) | powers `Get-Help` for your own functions |

### [02 · Desired State Configuration (DSC) Basics](level-3/02-dsc-basics.md)

| Concept | Purpose |
|---|---|
| `configuration { }` | declarative block compiled to one MOF per node |
| `Node "name" { }` | target a specific machine; one MOF file per node |
| Built-in resources (`File`, `Service`, `WindowsFeature`, `Registry`) | describe one piece of desired state |
| `-OutputPath` | where compiled `.mof` files land |
| `Start-DscConfiguration -Wait -Verbose` | apply a compiled configuration |
| `Test-DscConfiguration -Detailed` | audit drift without changing anything |
| `DependsOn = "[Type]Name"` | force one resource to apply after another |
| `Get-DscConfiguration` | show current state as last applied |

### [03 · Working with .NET from PowerShell](level-3/03-working-with-dotnet.md)

| Syntax | Meaning |
|---|---|
| `[Type]::Member` | access a static property/method |
| `[Type]::new(...)` | construct an instance (preferred over `New-Object`) |
| `.Property`, `.Method()` | instance members, same as any PowerShell object |
| `[void]$x.Method()` | call a method and discard its return value |
| `[System.Collections.Generic.List[T]]` | strongly-typed, faster list |
| `[System.Collections.Generic.Dictionary[K,V]]` | strongly-typed key/value store |
| `$_.Exception.GetType().Name` | the specific .NET exception type in a `catch` |
| `class` with a wrapped .NET member | compose .NET behavior you can't subclass directly |

### [04 · Building Reusable Modules](level-3/04-building-reusable-modules.md)

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

### [05 · Testing Advanced (Pester Mocking)](level-3/05-testing-advanced-mocking.md)

| Concept | Purpose |
|---|---|
| `Mock Cmd -ModuleName X` | mock a command as seen from inside module `X` |
| `Should -Invoke Cmd -ModuleName X` | assert against a mock defined with the same `-ModuleName` |
| `-ParameterFilter { ... }` | mock/assert conditionally, based on the arguments passed |
| `Should -Invoke ... -Times 0 -Exactly` | prove a side effect did NOT happen |
| `Should -Invoke ... -Times N -Exactly` | prove a call happened exactly N times, no more |
| `BeforeEach { Mock ... }` | fresh, reset mocks for every `It` |
| `-Throw "*substring*"` | assert an exception message contains specific text |

### [06 · Security](level-3/06-security.md)

| Concept | Use |
|---|---|
| `SecureString` | encrypted-in-memory string; damage limitation, not unbreakable |
| `[PSCredential]::new($user, $secureString)` | pair a username with a secure password |
| `$cred.GetNetworkCredential().Password` | decrypt only at the point of use |
| `ConvertTo-SecureString -AsPlainText -Force` on a literal | smell — hardcoded secret in disguise |
| `[Runtime.InteropServices.Marshal]::ZeroFreeBSTR` | explicitly clear decrypted memory after use |
| `Get-FileHash -Algorithm SHA256` | integrity checks — never password storage |
| `Select-String -Pattern 'password\|apikey\|secret'` | quick pre-commit secret sweep |
| `ExecutionPolicy` | accident prevention, not an attacker-facing security boundary |

### [07 · Performance Optimization](level-3/07-performance-optimization.md)

| Pattern | Verdict |
|---|---|
| `$array += $item` in a loop | avoid — O(n²), full copy every iteration |
| `[List[T]]::new()` + `.Add()` | fast, in-place growth |
| loop output captured by `$result = for (...) { }` | fast, often the simplest fix |
| `\|` pipeline over an in-memory collection | slower per-item, but streams (low memory) |
| `foreach` / `.Where()` / `.ForEach()` over in-memory data | faster for pure CPU-bound work |
| `Format-*`, `Out-GridView` inside a loop | avoid — format once, after the loop |
| `Measure-Command { }` | compare two implementations directly, run several times |

### [08 · Active Directory &amp; System Admin Cmdlets](level-3/08-active-directory-admin.md)

| Cmdlet | Purpose |
|---|---|
| `Get-ADUser -Filter "..." -Properties ...` | query users; `-Properties` needed beyond defaults |
| `-Filter` (PS syntax) vs `-LDAPFilter` (raw LDAP) | different grammars, don't mix |
| `New-ADUser ... -Enabled $true` | create; disabled by default without this |
| `Set-ADUser` / `Set-ADAccountPassword` | modify attributes / reset password |
| `Add-ADGroupMember` / `Remove-ADGroupMember -Confirm:$false` | manage group membership |
| `Get-ADComputer -Filter ...` | query computer objects |
| `New-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $true` | create OU with delete protection |
| `-WhatIf` | dry-run any state-changing AD cmdlet before running for real |

### [09 · Scheduled Tasks &amp; Automation](level-3/09-scheduled-tasks-automation.md)

| Tool | Platform | Purpose |
|---|---|---|
| `Register-ScheduledTask` + Action/Trigger/Settings | Windows | schedule a recurring script |
| `Get-ScheduledTaskInfo` | Windows | last run time/result, next run time |
| `crontab -e` | Linux/macOS | schedule `pwsh -File ...` |
| `Start-Job` / `Wait-Job` / `Receive-Job` / `Remove-Job` | cross-platform | background execution in a separate process |
| `-ArgumentList` / `$using:var` | cross-platform | pass data into a job (no automatic variable inheritance) |
| Retry-with-backoff pattern | cross-platform | survive transient failures unattended |
| `$PSScriptRoot` for all paths | cross-platform | scheduler working directory isn't guaranteed |

## Level 4 · Master

### [01 · Advanced Remoting &amp; Background Jobs](level-4/01-advanced-remoting-jobs.md)

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

### [02 · CI/CD with PowerShell](level-4/02-cicd-powershell.md)

| Piece | Purpose |
|---|---|
| `build.ps1 -Task Build\|Test\|All` | one entry point, same on a laptop and in CI |
| `New-PesterConfiguration` | explicit, discoverable test-run settings |
| `$config.Run.Exit = $true` | makes Pester set the process exit code on failure |
| Checking `$result.FailedCount` | connects test failure to pipeline failure |
| `shell: pwsh` in GitHub Actions | run YAML `run:` blocks as PowerShell |
| `runs-on: ubuntu-latest` | cheaper, cross-platform-safe unless Windows-only cmdlets are needed |
| Case-sensitive paths on Linux CI | common source of "works on my machine" |
| `exit 1` on any caught error | explicit pipeline failure signal beyond Pester's own |

### [03 · Cross-Platform PowerShell](level-4/03-cross-platform.md)

| Concern | Cross-platform fix |
|---|---|
| Which OS is this? | `$IsWindows` / `$IsLinux` / `$IsMacOS` |
| Building a path | `Join-Path`, never a literal `\` or `/` |
| Home directory | `$HOME` (works everywhere; avoid `$env:USERPROFILE`/`$env:HOME` directly) |
| Splitting text into lines | `-split '\r?\n'`, not just `"`n"` or `"`r`n"` |
| Filesystem case sensitivity | assume case-sensitive always, regardless of what your dev machine tolerates |
| Native command success/failure | check `$LASTEXITCODE`, not `$?` |
| `ls`, `dir`, other aliases | call `Get-ChildItem` directly in portable scripts |
| Environment variables | `$env:PATH` uses `[System.IO.Path]::PathSeparator` to split (`:` vs `;`) |

### [04 · Security Hardening (JEA)](level-4/04-security-hardening-jea.md)

| Piece | Purpose |
|---|---|
| `.psrc` (role capability file) | allow-list of cmdlets/functions/programs for one role |
| `.pssc` (session configuration file) | binds role(s) to security group(s), sets session type |
| `SessionType RestrictedRemoteServer` | minimal baseline, nothing extra unless granted |
| `RunAsVirtualAccount` | connecting user's identity never runs the commands |
| `RoleDefinitions` | maps a security group to a role capability |
| Parameter-constrained `VisibleCmdlets` | narrows a broad cmdlet to specific safe arguments |
| `Register-PSSessionConfiguration` | activates the endpoint (Windows/WinRM) |
| Testing as the restricted role | the only real verification of what's actually exposed |

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

| Concept | Purpose |
|---|---|
| `-Tag "Name"` on `Describe`/`It` | group tests for selective execution |
| `$config.Filter.Tag = 'Unit'` | run only tests matching a tag |
| `$config.TestResult.OutputFormat = 'NUnitXml'` | CI-parseable results |
| `$config.CodeCoverage.Enabled = $true` | measure which commands executed |
| `CoveragePercentTarget` | fail the build if coverage drops below a threshold |
| Command coverage ≠ scenario coverage | 100% coverage can still miss edge cases |
| Split suites by tag (`Unit` vs `Slow`/`Integration`) | keep fast feedback fast |

### [06 · Performance at Scale](level-4/06-performance-at-scale.md)

| Situation | Prefer |
|---|---|
| Repeated membership/lookup checks against a large, static set | hashtable or `HashSet[T]`, built once |
| One-off `-contains` check on a small array | plain array is fine, don't over-engineer |
| Reading a very large file, need speed and low memory | `[System.IO.File]::ReadLines()` |
| Reading a large file, readability matters more than raw speed | `Get-Content` (streams, pipeline-friendly) |
| Need the whole file as one string (regex across lines, etc.) | `Get-Content -Raw` (accept the memory cost) |
| Need only the first N results (or first match) | keep it in the pipeline, use `-First`/short-circuit `Where-Object` |
| Any performance claim | verify with `Measure-Command` on your actual data, don't assume |

### [07 · Building PowerShell-based DevOps Tools](level-4/07-devops-tools.md)

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

### [08 · Publishing Modules (PowerShell Gallery)](level-4/08-publishing-modules.md)

| Step | Tool |
|---|---|
| Generate a correct manifest | `New-ModuleManifest` |
| Validate it parses correctly | `Test-ModuleManifest` |
| Lint before publishing | `Invoke-ScriptAnalyzer -Recurse` |
| Publish | `Publish-Module -Path ... -NuGetApiKey ...` |
| Bump version before every publish | `ModuleVersion` in `.psd1`, semantic versioning |
| Declare PowerShell/module dependencies | `#Requires`, `RequiredModules` in manifest |
| Keep the publish folder clean | scope `-Path` to only real module content |

### [09 · Logging &amp; Observability for Scripts](level-4/09-logging-observability.md)

| Need | Tool |
|---|---|
| Structured, machine-parseable entries | custom `Write-Log` writing JSON Lines |
| Time an operation, capture success/failure | `[System.Diagnostics.Stopwatch]` + `try`/`catch` |
| Capture an entire interactive session verbatim | `Start-Transcript` / `Stop-Transcript` |
| Make non-terminating errors catchable | `$ErrorActionPreference = 'Stop'` or `-ErrorAction Stop` |
| Find where an error happened | `$_.InvocationInfo.ScriptLineNumber` |
| Avoid the tight-loop logging trap | batch entries, write once instead of per-iteration |

### [10 · Capstone Project](level-4/10-capstone-project.md)

| Concept | Where it came from |
|---|---|
| Public/Private folder split | Building Reusable Modules (Level 3) |
| Generated manifest, `Test-ModuleManifest` | Publishing Modules (Level 4, module 08) |
| `-ErrorAction SilentlyContinue` for expected absence | Error handling (Level 1-2) |
| `[System.Diagnostics.Stopwatch]` timing | Performance at Scale (Level 4, module 06) |
| Structured JSON-lines logging | Logging & Observability (Level 4, module 09) |
| Tagged Pester tests, `Unit` vs `Integration` | Testing at Scale & CI (Level 4, module 05) |
| Cross-platform `Get-PSDrive` | Cross-Platform PowerShell (Level 4, module 03) |

---

Go deeper with the full [PowerShell Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
