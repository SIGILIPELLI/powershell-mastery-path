---
title: "PowerShell Interview Questions and Answers (179 Questions)"
description: "179 PowerShell interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the PowerShell Mastery Path."
---

# PowerShell Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free PowerShell Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Script](level-1/01-setup.md)

**Q1.** What makes PowerShell different from traditional shells?

**Answer:** Cmdlets pass .NET objects through the pipeline, not just text

Objects with properties and methods.

**Q2.** What naming pattern do cmdlets follow?

**Answer:** Verb-Noun (e.g. Get-Process)

Approved verbs via Get-Verb.

**Q3.** Which cmdlet helps you discover commands?

**Answer:** Get-Command (and Get-Help for usage)

Update-Help first.

**Q4.** Which cmdlet shows an object's properties and methods?

**Answer:** Get-Member

Inspect pipeline output.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```powershell
$a = 5; $b = "5"; $c = 2.5
"$($a -eq $b) $($b -eq $a) $($a.GetType().Name) $($c.GetType().Name) $([int]"12" + 1)"
```

**Answer:** True True Int32 Double 13

-eq coerces the right side to the left type. Verified by running it in PowerShell 7.

**Q6.** What does this print?

```powershell
[string]$s = 42
$n = $null
"$($s.GetType().Name) [$n] $($n -eq $null) $('abc' -like 'a*') $("Hi $s")"
```

**Answer:** String [] True True Hi 42

Typed variables convert on assignment. Verified by running it in PowerShell 7.

**Q7.** What does `$null -eq $x` vs `$x -eq $null` matter for?

**Answer:** With arrays, put $null on the left so the comparison isn't a filter

Array -eq filters.

**Q8.** Which quoting expands variables?

**Answer:** Double quotes (single quotes are literal)

"$x" vs '$x'.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```powershell
$out = foreach ($i in 1..6) { if ($i % 2) { continue }; $i * 10 }
$out -join ","
switch (5) { {$_ -gt 3} { "big" } {$_ -gt 1} { "medium" } default { "small" } }
```

**Answer:** 20,40,60
big
medium

switch runs every matching clause unless break. Verified by running it in PowerShell 7.

**Q10.** What does this print?

```powershell
$i = 0
do { $i++ } while ($i -lt 3)
$j = 10
while ($j -gt 7) { $j-- }
"$i $j " + ($true ? "x" : "y")
```

**Answer:** 3 7 x

do/while runs at least once; the ternary needs PowerShell 7. Verified by running it in PowerShell 7.

**Q11.** Which operator is the ternary in PowerShell 7?

**Answer:** $cond ? 'a' : 'b'

PS 7+.

**Q12.** What does `break` do inside switch?

**Answer:** Stops evaluating further matching clauses

Otherwise all matches run.

### [04 · Functions](level-1/04-functions.md)

**Q13.** What does this print?

```powershell
function Get-Area {
    [CmdletBinding()]
    param([Parameter(Mandatory)][double]$Width, [double]$Height = 2)
    $Width * $Height
}
(Get-Area -Width 3) + (Get-Area 4 5)
```

**Answer:** 26

Defaults and positional parameters. Verified by running it in PowerShell 7.

**Q14.** What does this print?

```powershell
function f { param($x) return "got $x" ; "never" }
f 7
function g { "a"; "b"; 3 }
(g).Count
```

**Answer:** got 7
3

Everything not captured is output. Verified by running it in PowerShell 7.

**Q15.** What does [CmdletBinding()] add?

**Answer:** Common parameters like -Verbose and -ErrorAction and advanced function behavior

Makes it cmdlet-like.

**Q16.** What does `return` do besides emit a value?

**Answer:** Exits the function immediately

Later statements don't run.

### [05 · Working with Objects &amp; the Pipeline](level-1/05-objects-pipeline.md)

**Q17.** What does this print?

```powershell
$p = [pscustomobject]@{Name='a'; Size=30}, [pscustomobject]@{Name='b'; Size=10}, [pscustomobject]@{Name='c'; Size=20}
($p | Where-Object Size -gt 15 | Sort-Object Size | ForEach-Object Name) -join ","
($p | Measure-Object Size -Sum -Average).Sum
($p | Select-Object -First 1).Name
```

**Answer:** c,a
60
a

Objects flow through the pipeline. Verified by running it in PowerShell 7.

**Q18.** What does this print?

```powershell
$p = 1..10 | Group-Object { $_ % 3 }
($p | ForEach-Object { "$($_.Name):$($_.Count)" }) -join " "
(1..5 | Select-Object -Last 2) -join ","
(Get-Date '2026-01-31').AddMonths(1).ToString('yyyy-MM-dd')
```

**Answer:** 0:3 1:4 2:3
4,5
2026-02-28

Group-Object and date arithmetic. Verified by running it in PowerShell 7.

**Q19.** What does Select-Object -Property do?

**Answer:** Creates new objects with chosen properties (or calculated ones)

Projection.

**Q20.** When prefer Where-Object -Property syntax to a script block?

**Answer:** For simple comparisons; it's concise and faster to read

PS 3+ syntax.

### [06 · Arrays &amp; Hashtables](level-1/06-arrays-hashtables.md)

**Q21.** What does this print?

```powershell
$a = 1,2,3
$a += 4
$h = @{b=2; a=1}
$h['c'] = 3
"$($a.Count) $($h.Count) $(($h.Keys | Sort-Object) -join '') $($a[-1]) $($h.ContainsKey('z'))"
```

**Answer:** 4 3 abc 4 False

Arrays are fixed size: += creates a new array. Verified by running it in PowerShell 7.

**Q22.** What does this print?

```powershell
$o = [ordered]@{x=1; y=2}
$o.Keys -join ","
$s = @{a=1}.GetEnumerator() | ForEach-Object { "$($_.Key)=$($_.Value)" }
$s
@(1,2,3)[1..2] -join ","
```

**Answer:** x,y
a=1
2,3

Ordered hashtables keep insertion order. Verified by running it in PowerShell 7.

**Q23.** Why use a generic List instead of += on big arrays?

**Answer:** += copies the entire array each time (O(n²))

System.Collections.Generic.List.

**Q24.** How splat parameters?

**Answer:** Put them in a hashtable and pass @params

Readable calls.

### [07 · Working with Files](level-1/07-working-with-files.md)

**Q25.** What does this print?

```powershell
$d = Join-Path ([IO.Path]::GetTempPath()) "pstest$(Get-Random)"
New-Item -ItemType Directory $d | Out-Null
'one','two' | Set-Content "$d/a.txt"
Add-Content "$d/a.txt" 'three'
(Get-Content "$d/a.txt").Count
(Get-Content "$d/a.txt" -Raw).Length
(Get-ChildItem $d).Name
Remove-Item $d -Recurse
```

**Answer:** 3
14
a.txt

Set-Content writes lines; -Raw returns one string (with newlines). Verified by running it in PowerShell 7.

**Q26.** Which parameter treats a path literally (no wildcards)?

**Answer:** -LiteralPath

Brackets in names.

**Q27.** What does Test-Path return?

**Answer:** True or False for whether the path exists

Guard file operations.

**Q28.** Which encoding does PowerShell 7 write by default?

**Answer:** UTF-8 without BOM

Windows PowerShell 5.1 differs.

### [08 · Error Handling Basics](level-1/08-error-handling-basics.md)

**Q29.** What does this print?

```powershell
try { 1/0 } catch { "caught: $($_.Exception.GetType().Name)" } finally { "done" }
try { Get-Item /nonexistent_zzz -ErrorAction Stop } catch { "stop: " + $_.CategoryInfo.Category }
Get-Item /nonexistent_zzz -ErrorAction SilentlyContinue
"after"
```

**Answer:** caught: RuntimeException
done
stop: ObjectNotFound
after

Non-terminating errors need -ErrorAction Stop to be caught. Verified by running it in PowerShell 7.

**Q30.** What does this print?

```powershell
$ErrorActionPreference = 'Stop'
try { Get-Item /nonexistent_zzz; "not here" } catch { "handled" }
$Error.Count -ge 1
```

**Answer:** handled
True

Preference variable affects the scope. Verified by running it in PowerShell 7.

**Q31.** What's the difference between terminating and non-terminating errors?

**Answer:** Terminating errors stop execution and are caught; non-terminating ones continue unless made terminating

-ErrorAction Stop.

**Q32.** What does $? hold?

**Answer:** Whether the last command succeeded

$LASTEXITCODE for native.

### [09 · Modules Basics](level-1/09-modules-basics.md)

**Q33.** What does Import-Module do?

**Answer:** Loads a module's commands into the session

Auto-loading often handles it.

**Q34.** Where do you find modules in the gallery?

**Answer:** Find-Module / Find-PSResource (PowerShell Gallery)

Install-PSResource in newer versions.

**Q35.** What does $env:PSModulePath contain?

**Answer:** Folders searched for modules

Auto-discovery.

**Q36.** Why review modules before installing?

**Answer:** They run code with your privileges; check publisher and source

Supply chain.

### [10 · Project — System Info Reporter](level-1/10-project-system-info-reporter.md)

**Q37.** What does the system info reporter combine?

**Answer:** Cmdlets, objects, formatting and file output

Level 1 project.

**Q38.** Which cmdlet exports objects to a file as CSV?

**Answer:** Export-Csv

-NoTypeInformation is default in 7.

**Q39.** Why output objects rather than formatted text from your script?

**Answer:** Callers can filter, sort and export them; format only at the end

Format-* last.

**Q40.** How make the report cross-platform?

**Answer:** Use cmdlets/properties available on all OSes and test on each

Check $IsWindows.

## Level 2 · Intermediate

### [01 · Advanced Pipeline](level-2/01-advanced-pipeline.md)

**Q41.** What does this print?

```powershell
1..4 | ForEach-Object { "in$_"; $_ } | ForEach-Object { if ($_ -is [int]) { $_ * 2 } }
```

**Answer:** 2
4
6
8

Strings are filtered out by the type test; only ints are doubled. Verified by running it in PowerShell 7.

**Q42.** What does this print?

```powershell
$r = 1..5 | ForEach-Object -Begin { $sum = 0 } -Process { $sum += $_ } -End { "total=$sum" }
$r
(1..10 | Where-Object { $_ % 2 -eq 0 } | Select-Object -First 2) -join ","
```

**Answer:** total=15
2,4

Begin/Process/End blocks. Verified by running it in PowerShell 7.

**Q43.** Why is Select-Object -First N efficient on streams?

**Answer:** It stops the upstream pipeline once N objects arrive

Early exit.

**Q44.** Which cmdlets must buffer all input?

**Answer:** Sort-Object and Group-Object

Memory cost.

### [02 · Cmdlet Design](level-2/02-cmdlet-design.md)

**Q45.** What does this print?

```powershell
function Get-Square {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][int[]]$Number)
    process { foreach ($n in $Number) { [pscustomobject]@{ Number = $n; Square = $n * $n } } }
}
(1..3 | Get-Square | ForEach-Object { "$($_.Number)^2=$($_.Square)" }) -join " "
```

**Answer:** 1^2=1 2^2=4 3^2=9

process{} runs per pipeline input. Verified by running it in PowerShell 7.

**Q46.** Which block runs once per pipeline object?

**Answer:** process

begin/process/end.

**Q47.** What does SupportsShouldProcess add?

**Answer:** -WhatIf and -Confirm support for changing commands

Safety.

**Q48.** What should a cmdlet-style function emit?

**Answer:** Objects (not formatted text)

Composable.

### [03 · Error Handling Advanced](level-2/03-error-handling-advanced.md)

**Q49.** What does this print?

```powershell
try {
    try { throw [System.IO.FileNotFoundException]::new("missing.txt") }
    catch [System.IO.FileNotFoundException] { throw "wrapped: $($_.Exception.Message)" }
} catch { $_.Exception.Message }
$ErrorActionPreference = 'Continue'
$null = Get-Item /zzz_none -ErrorVariable ev -ErrorAction SilentlyContinue
$ev.Count
```

**Answer:** wrapped: missing.txt
1

Typed catch, rethrow, ErrorVariable. Verified by running it in PowerShell 7.

**Q50.** What does -ErrorAction SilentlyContinue do?

**Answer:** Suppresses non-terminating error display but still records it in $Error

Not a try/catch.

**Q51.** What does `throw` produce?

**Answer:** A terminating error

Caught by catch.

**Q52.** Why use finally?

**Answer:** Cleanup that must run whether or not an error occurred

Close handles.

### [04 · Regular Expressions in PowerShell](level-2/04-regular-expressions.md)

**Q53.** What does this print?

```powershell
$text = "Order #1234 shipped 2026-03-15 to ann@example.com"
$m = [regex]::Match($text, '#(?<id>\d+).*?(?<date>\d{4}-\d{2}-\d{2})')
"$($m.Groups['id'].Value) $($m.Groups['date'].Value)"
$text -match '(\w+)@(\w+)\.com'; $Matches[1] + "/" + $Matches[2]
"a1b22c333" -replace '\d+', '#'
([regex]::Matches("a1b22c333", '\d+') | ForEach-Object Value) -join "|"
```

**Answer:** 1234 2026-03-15
True
ann/example
a#b#c#
1|22|333

Named groups and -replace. Verified by running it in PowerShell 7.

**Q54.** Which operator tests a regex match and fills $Matches?

**Answer:** -match

Case-insensitive by default.

**Q55.** How make matching case-sensitive?

**Answer:** Use -cmatch

c prefix.

**Q56.** What is the difference between -like and -match?

**Answer:** -like uses wildcards; -match uses regular expressions

Choose per need.

### [05 · Remoting Basics](level-2/05-remoting-basics.md)

**Q57.** Which cmdlet runs commands on a remote machine?

**Answer:** Invoke-Command (over WinRM or SSH)

Remoting.

**Q58.** What does Enter-PSSession do?

**Answer:** Starts an interactive session on a remote computer

Exit-PSSession.

**Q59.** What do remote results look like?

**Answer:** Deserialized objects (properties but no live methods)

Remember when calling methods.

**Q60.** Which transport does cross-platform PowerShell 7 remoting commonly use?

**Answer:** SSH

WinRM on Windows.

### [06 · Testing with Pester](level-2/06-testing-pester.md)

**Q61.** What is Pester?

**Answer:** PowerShell's testing framework

Describe/It/Should.

**Q62.** What does Should -Be do?

**Answer:** Asserts a value equals the expected value

Assertions.

**Q63.** What does Mock do?

**Answer:** Replaces a command's behavior during a test

Isolation.

**Q64.** Which command runs the tests?

**Answer:** Invoke-Pester

Configuration objects.

### [07 · Working with JSON/REST APIs](level-2/07-json-rest-apis.md)

**Q65.** What does this print?

```powershell
$j = '{"name":"Ada","tags":["a","b"],"n":{"k":1.5}}'
$o = $j | ConvertFrom-Json
"$($o.name) $($o.tags.Count) $($o.n.k)"
[pscustomobject]@{x=1; y=@(1,2)} | ConvertTo-Json -Compress
@{a=@{b=1}} | ConvertTo-Json -Depth 1 -Compress
```

**Answer:** Ada 2 1.5
{"x":1,"y":[1,2]}
{"a":{"b":1}}

JSON round trip. Verified by running it in PowerShell 7.

**Q66.** Which cmdlet calls a REST API and parses JSON?

**Answer:** Invoke-RestMethod

Invoke-WebRequest returns raw response.

**Q67.** What controls nested JSON output?

**Answer:** ConvertTo-Json -Depth

Default depth 2.

**Q68.** How send a JSON body?

**Answer:** -Body ($obj | ConvertTo-Json) -ContentType 'application/json'

Headers too.

### [08 · Script Modules &amp; Manifests](level-2/08-script-modules-manifests.md)

**Q69.** What does a module manifest (.psd1) describe?

**Answer:** Version, author, exports and dependencies

New-ModuleManifest.

**Q70.** What does Export-ModuleMember do?

**Answer:** Controls which functions are visible outside the module

Public API.

**Q71.** What's the file extension of a script module?

**Answer:** *.psm1

Loaded by Import-Module.

**Q72.** Why put private helpers in the module but not export them?

**Answer:** Keeps the public surface small and stable

Encapsulation.

### [09 · Working with CSV/Excel Data](level-2/09-csv-excel-data.md)

**Q73.** What does this print?

```powershell
$csv = @"
name,dept,salary
ann,eng,100
bob,ops,80
cy,eng,120
"@
$rows = $csv | ConvertFrom-Csv
($rows | Group-Object dept | ForEach-Object { "$($_.Name)=$(($_.Group | Measure-Object salary -Sum).Sum)" }) -join " "
($rows | Sort-Object { [int]$_.salary } -Descending | Select-Object -First 1).name
$rows[0].salary.GetType().Name
```

**Answer:** eng=220 ops=80
cy
String

CSV values are strings unless cast. Verified by running it in PowerShell 7.

**Q74.** Why not export CSV from formatted output?

**Answer:** Format-* output is for display; export the objects

Format-Table breaks it.

**Q75.** How read Excel without Excel installed?

**Answer:** Use the ImportExcel module

Gallery module.

**Q76.** What does Import-Csv -Delimiter do?

**Answer:** Reads files with other separators like ; or tab

Locale CSVs.

### [10 · Project — Log Monitoring Script](level-2/10-project-log-monitoring.md)

**Q77.** What does the log monitoring project combine?

**Answer:** Regex parsing, grouping, thresholds, CSV/JSON reports and tests

Level 2 project.

**Q78.** How stream a big log efficiently?

**Answer:** Get-Content | Where-Object or Select-String instead of loading all lines

Streaming.

**Q79.** Which cmdlet searches text files with regex?

**Answer:** Select-String

-Pattern.

**Q80.** How avoid duplicate alerts?

**Answer:** Track state (e.g. last offset or timestamp) between runs

Idempotence.

## Level 3 · Advanced

### [01 · Advanced Functions &amp; Parameter Validation](level-3/01-advanced-functions-validation.md)

**Q81.** What does this print?

```powershell
function Set-Level {
    [CmdletBinding()]
    param(
        [ValidateSet('low','mid','high')][string]$Level = 'mid',
        [ValidateRange(1,10)][int]$Count = 1,
        [ValidatePattern('^[a-z]+$')][string]$Name = 'x'
    )
    "$Level $Count $Name"
}
Set-Level -Level high -Count 3 -Name abc
try { Set-Level -Count 11 } catch { "invalid" }
try { Set-Level -Level nope } catch { "invalid" }
```

**Answer:** high 3 abc
invalid
invalid

Validation attributes reject bad input before the body runs. Verified by running it in PowerShell 7.

**Q82.** What does ValidateNotNullOrEmpty do?

**Answer:** Rejects null or empty values

Common guard.

**Q83.** What are parameter sets for?

**Answer:** Mutually exclusive groups of parameters

DefaultParameterSetName.

**Q84.** Why use ValueFromPipelineByPropertyName?

**Answer:** Bind parameters from matching property names on piped objects

Composable commands.

### [02 · Desired State Configuration (DSC) Basics](level-3/02-dsc-basics.md)

**Q85.** What is DSC?

**Answer:** Desired State Configuration: declare the state; the engine makes it so

Idempotent.

**Q86.** What is idempotence?

**Answer:** Applying a configuration repeatedly gives the same result without extra changes

Key concept.

**Q87.** What does Test-DscConfiguration/Test-TargetResource do?

**Answer:** Reports whether the machine matches the desired state

Check before set.

**Q88.** What's a modern alternative for cross-platform config?

**Answer:** Ansible, Azure Automanage/Guest Config, or DSC v3

Choose per environment.

### [03 · Working with .NET from PowerShell](level-3/03-working-with-dotnet.md)

**Q89.** What does this print?

```powershell
$sb = [System.Text.StringBuilder]::new()
[void]$sb.Append("a").Append("b")
$sb.ToString()
[math]::Round(2.5), [math]::Round(3.5), [math]::Pow(2,10), [Math]::Floor(-1.5)
[datetime]::new(2026,3,15).DayOfWeek
[guid]::Empty.ToString().Length
```

**Answer:** ab
2
4
1024
-2
Sunday
36

Banker's rounding; static .NET members. Verified by running it in PowerShell 7.

**Q90.** How call a static .NET method?

**Answer:** [Namespace.Type]::Method()

Double colon.

**Q91.** How load an extra .NET assembly?

**Answer:** Add-Type (-AssemblyName or -Path)

Or `using assembly`.

**Q92.** Why use .NET types sparingly?

**Answer:** Cmdlets are more discoverable and cross-platform-safe; use .NET for gaps and performance

Balance.

### [04 · Building Reusable Modules](level-3/04-building-reusable-modules.md)

**Q93.** What makes a module reusable?

**Answer:** Clear public functions, help, validation, tests and a manifest

Design for others.

**Q94.** What should functions avoid?

**Answer:** Writing to the host and relying on global state

Return objects.

**Q95.** What does comment-based help provide?

**Answer:** Get-Help documentation from comments above functions

.SYNOPSIS/.EXAMPLE.

**Q96.** How version modules?

**Answer:** Semantic versioning in the manifest

Breaking changes bump major.

### [05 · Testing Advanced (Pester Mocking)](level-3/05-testing-advanced-mocking.md)

**Q97.** What does this print?

```powershell
Remove-Module Pester -ErrorAction SilentlyContinue
function Get-Greeting { param($Name) "hi $Name" }
function Say { Get-Greeting -Name 'x' }
function Get-Greeting { param($Name) "mock $Name" }
Say
```

**Answer:** mock x

Redefining a function shadows the earlier one (mocking in spirit). Verified by running it in PowerShell 7.

**Q98.** What does Should -Invoke verify?

**Answer:** That a mocked command was called (and how many times)

Interaction checks.

**Q99.** What is BeforeAll/BeforeEach for?

**Answer:** Setup shared by tests (once vs per test)

Fixtures.

**Q100.** What goes wrong with over-mocking?

**Answer:** Tests mirror implementation and break on refactors

Mock boundaries only.

### [06 · Security](level-3/06-security.md)

**Q101.** What does Set-ExecutionPolicy control?

**Answer:** Which scripts may run (a safety feature, not a security boundary)

Windows.

**Q102.** How store credentials securely?

**Answer:** SecretManagement/SecretStore or a vault; never in plain text

PSCredential objects.

**Q103.** What does Get-Credential return?

**Answer:** A PSCredential with the password as a SecureString

Prompt.

**Q104.** What should you do with logs?

**Answer:** Avoid writing secrets or tokens

Redact.

### [07 · Performance Optimization](level-3/07-performance-optimization.md)

**Q105.** What does this print?

```powershell
$n = 20000
$a = 1..$n | ForEach-Object { $_ }
$l = [System.Collections.Generic.List[int]]::new()
foreach ($i in 1..$n) { $l.Add($i) }
"$($a.Count) $($l.Count) $(($a | Measure-Object -Sum).Sum -eq ($l | Measure-Object -Sum).Sum)"
$swa = (Measure-Command { foreach ($i in 1..1000) { $null = $i * 2 } }).TotalMilliseconds -ge 0
$swa
```

**Answer:** 20000 20000 True
True

Both approaches produce the same data; the List avoids array copying. Verified by running it in PowerShell 7.

**Q106.** What is the fastest way to build a large collection in a loop?

**Answer:** A generic List (Add) or assign the loop output directly

Avoid += on arrays.

**Q107.** How measure code speed?

**Answer:** Measure-Command (repeat and compare)

Profile.

**Q108.** Why filter left in a pipeline?

**Answer:** Reduce objects early so later stages do less work

-Filter on providers.

### [08 · Active Directory &amp; System Admin Cmdlets](level-3/08-active-directory-admin.md)

**Q109.** Which module manages Active Directory?

**Answer:** ActiveDirectory (RSAT) with Get-ADUser etc.

Windows.

**Q110.** Why use -Filter with Get-ADUser?

**Answer:** Server-side filtering is far faster than fetching all users

Don't Where-Object everything.

**Q111.** What should bulk AD changes use first?

**Answer:** -WhatIf and a pilot run

Safe change.

**Q112.** How limit property retrieval?

**Answer:** -Properties with only the needed attributes

Performance.

### [09 · Scheduled Tasks &amp; Automation](level-3/09-scheduled-tasks-automation.md)

**Q113.** Which cmdlets register scheduled tasks on Windows?

**Answer:** Register-ScheduledTask with New-ScheduledTaskAction/Trigger

Windows only.

**Q114.** What should scheduled scripts do?

**Answer:** Log, handle errors, use least privilege and be idempotent

Unattended.

**Q115.** How run something on Linux on a schedule?

**Answer:** cron or systemd timers calling pwsh

Cross-platform.

**Q116.** How prevent overlapping runs?

**Answer:** A lock file/mutex or checking for a running instance

Reliability.

### [10 · Project — Admin Toolkit Module](level-3/10-project-admin-toolkit.md)

**Q117.** What does the admin toolkit project combine?

**Answer:** Reusable module, validation, logging, tests, remoting and scheduling

Level 3 project.

**Q118.** How structure the toolkit?

**Answer:** A module with public/private functions, manifest and Pester tests

Maintainable.

**Q119.** How make it safe to run?

**Answer:** -WhatIf/-Confirm, validation, least privilege and logging

Defence.

**Q120.** How share it?

**Answer:** Publish to a private repository/feed or the PowerShell Gallery

Versioned.

## Level 4 · Master

### [01 · Advanced Remoting &amp; Background Jobs](level-4/01-advanced-remoting-jobs.md)

**Q121.** What does Start-Job do?

**Answer:** Runs a script block in a background job

Receive-Job gets output.

**Q122.** What does ForEach-Object -Parallel do (PS 7)?

**Answer:** Runs the script block for items in parallel with -ThrottleLimit

Use $using: for variables.

**Q123.** When is parallelism not worth it?

**Answer:** For tiny tasks where startup overhead dominates

Measure.

**Q124.** How handle remoting at scale?

**Answer:** Invoke-Command to many hosts with -ThrottleLimit and error collection

Fan-out.

### [02 · CI/CD with PowerShell](level-4/02-cicd-powershell.md)

**Q125.** What should a PowerShell CI pipeline run?

**Answer:** Pester tests, PSScriptAnalyzer and module build/publish steps

Quality gates.

**Q126.** What does PSScriptAnalyzer do?

**Answer:** Static analysis for best-practice and style issues

Invoke-ScriptAnalyzer.

**Q127.** How store pipeline secrets?

**Answer:** CI secret variables, never in the repo

Security.

**Q128.** What artifacts help?

**Answer:** Test results and code coverage reports

Visibility.

### [03 · Cross-Platform PowerShell](level-4/03-cross-platform.md)

**Q129.** What does this print?

```powershell
$IsWindows -or $IsLinux -or $IsMacOS
[System.IO.Path]::DirectorySeparatorChar.ToString().Length
Join-Path 'a' 'b' | ForEach-Object { $_ -replace '[\\/]', '/' }
$PSVersionTable.PSEdition
```

**Answer:** True
1
a/b
Core

PowerShell 7 is Core and cross-platform. Verified by running it in PowerShell 7.

**Q130.** Which cmdlet builds paths safely across OSes?

**Answer:** Join-Path

Separators.

**Q131.** What's a cross-platform pitfall?

**Answer:** Case-sensitive file systems and Windows-only cmdlets/modules

Test on each OS.

**Q132.** How check the platform?

**Answer:** $IsWindows / $IsLinux / $IsMacOS

Automatic variables.

### [04 · Security Hardening (JEA)](level-4/04-security-hardening-jea.md)

**Q133.** What is JEA?

**Answer:** Just Enough Administration: constrained remoting endpoints exposing only allowed commands

Least privilege.

**Q134.** What defines a JEA endpoint?

**Answer:** A role capability file and a session configuration

.psrc/.pssc.

**Q135.** What does constrained language mode do?

**Answer:** Restricts risky language features to reduce attack surface

Security.

**Q136.** What logging detects abuse?

**Answer:** Script block, module and transcription logging

Audit.

### [05 · Testing at Scale &amp; CI](level-4/05-testing-at-scale-ci.md)

**Q137.** How scale test suites in CI?

**Answer:** Parallel jobs, tags and cached modules

Faster feedback.

**Q138.** What should tests avoid?

**Answer:** Dependence on real servers or clocks (mock boundaries)

Deterministic.

**Q139.** How report results?

**Answer:** NUnit/JUnit XML consumed by CI

Dashboards.

**Q140.** What do you do with flaky tests?

**Answer:** Fix isolation/timing or quarantine with a ticket

Trust.

### [06 · Performance at Scale](level-4/06-performance-at-scale.md)

**Q141.** Which technique saves memory on huge inputs?

**Answer:** Stream with the pipeline or Get-Content -ReadCount instead of loading everything

Batching.

**Q142.** What does -ReadCount do?

**Answer:** Sends lines to the pipeline in chunks

Throughput.

**Q143.** How avoid slow string concatenation in loops?

**Answer:** StringBuilder or -join

Quadratic cost.

**Q144.** What first?

**Answer:** Measure before optimizing

Evidence.

### [07 · Building PowerShell-based DevOps Tools](level-4/07-devops-tools.md)

**Q145.** Which tools do DevOps scripts often call?

**Answer:** git, docker, kubectl, az/aws CLIs and REST APIs

Glue.

**Q146.** How check a native command's success?

**Answer:** $LASTEXITCODE (and $? for cmdlets)

Native exit codes.

**Q147.** How capture native output as objects?

**Answer:** Convert text with ConvertFrom-Json or parse lines

--output json.

**Q148.** Why wrap native tools in functions?

**Answer:** Consistent errors, parameters and testability

Abstraction.

### [08 · Publishing Modules (PowerShell Gallery)](level-4/08-publishing-modules.md)

**Q149.** What do you need to publish a module?

**Answer:** A manifest, tests and an API key for the gallery/feed

Publish-Module/Publish-PSResource.

**Q150.** What belongs in a README/changelog?

**Answer:** Install steps, examples and version history

User docs.

**Q151.** How sign scripts?

**Answer:** Authenticode certificates (Set-AuthenticodeSignature)

Trust.

**Q152.** How version breaking changes?

**Answer:** Bump the major version and document migration

SemVer.

### [09 · Logging &amp; Observability for Scripts](level-4/09-logging-observability.md)

**Q153.** What makes logs useful?

**Answer:** Structured, levelled entries with correlation IDs and timestamps

Query later.

**Q154.** Which streams exist?

**Answer:** Output, Error, Warning, Verbose, Debug, Information

Redirect with 1&gt;..6&gt;.

**Q155.** What does Start-Transcript capture?

**Answer:** Console input and output to a file

Audit trail.

**Q156.** How emit metrics?

**Answer:** Write structured events to a log system or counters to monitoring

Observability.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** A tested, secured, published module with CI, logging and docs

Production-ready.

**Q158.** How prove readiness?

**Answer:** CI green on all platforms, security review and release notes

Evidence.

**Q159.** What documentation helps?

**Answer:** README, examples, runbooks and a contributing guide

Onboarding.

**Q160.** What sustains it?

**Answer:** Ownership, dependency updates and issue triage

Maintenance.

## Mixed questions across all levels

**Q161.** What does this print?

```powershell
$a = 1,2,3
$a -eq 2
$a -contains 2
$null -eq $a
(1..3 | ForEach-Object { $_ * $_ }) -join ","
```

**Answer:** 2
True
False
1,4,9

-eq on an array filters; -contains tests membership. Verified by running it in PowerShell 7.

**Q162.** What does this print?

```powershell
function f { param([int]$n) $n * 2 }
f "21"
try { f "abc" } catch { "bad" }
```

**Answer:** 42
bad

Typed parameters convert or throw. Verified by running it in PowerShell 7.

**Q163.** What does this print?

```powershell
$h = @{ a = 1; b = 2 }
$h.GetEnumerator() | Sort-Object Name | ForEach-Object { "$($_.Name)$($_.Value)" }
($h.Keys | Measure-Object).Count
```

**Answer:** a1
b2
2

Hashtable enumeration. Verified by running it in PowerShell 7.

**Q164.** What does this print?

```powershell
$ErrorActionPreference = 'Continue'
Get-Item /nope_zzz 2>$null
"next"
try { Get-Item /nope_zzz -ErrorAction Stop } catch { "caught" }
```

**Answer:** next
caught

Non-terminating errors continue; -ErrorAction Stop makes them catchable. Verified by running it in PowerShell 7.

**Q165.** What does this print?

```powershell
"{0:N1} {1:P0} {2:D3}" -f 3.14159, 0.256, 7
'a,b,,c'.Split(',').Count
('x' * 3) + 'y'
"Hello".Substring(1,3)
```

**Answer:** 3.1 26% 007
4
xxxy
ell

Format operator and string methods. Verified by running it in PowerShell 7.

**Q166.** What does this print?

```powershell
$o = [pscustomobject]@{ A = 1 }
$o | Add-Member -NotePropertyName B -NotePropertyValue 2
($o | Get-Member -MemberType NoteProperty).Name -join ","
$o.B * 5
```

**Answer:** A,B
10

Objects are extensible. Verified by running it in PowerShell 7.

**Q167.** Why prefer returning objects over Write-Host text?

**Answer:** Callers can filter, sort and export objects; host text is for display only

Composable scripts.

**Q168.** A script must run unattended nightly. What is essential?

**Answer:** Logging, error handling with non-zero exit codes and least-privilege credentials

Unattended reliability.

**Q169.** When does Get-Content huge.log | Select-Object -First 5 stop reading?

**Answer:** As soon as 5 lines are received (the pipeline signals upstream to stop)

Streaming.

**Q170.** Which cmdlet adds safety for destructive actions?

**Answer:** -WhatIf / -Confirm via SupportsShouldProcess

Preview first.

**Q171.** Remote results are missing methods. Why?

**Answer:** Objects are serialized and come back deserialized (properties only)

Call methods remotely.

**Q172.** A module's helper functions leak to users. Fix?

**Answer:** Export only public functions with Export-ModuleMember or the manifest FunctionsToExport

Small public API.

**Q173.** A native tool fails but the script continues. What do you check?

**Answer:** $LASTEXITCODE after calling it (and use throw)

Native commands don't throw.

**Q174.** Which Pester feature isolates a test from the network?

**Answer:** Mock the command that calls out

Deterministic tests.

**Q175.** How avoid secrets in scripts?

**Answer:** Use SecretManagement/vaults or CI secret variables

Least exposure.

**Q176.** Why use [ordered] for output records?

**Answer:** Predictable column order in reports and JSON

Readable output.

**Q177.** What's the main cost of $array += $item in a loop?

**Answer:** Each += allocates and copies a new array

Use List[T].

**Q178.** What is -Parallel good for in PowerShell 7?

**Answer:** Running independent, slow tasks (I/O or remote) concurrently

Use ThrottleLimit.

**Q179.** What belongs in a published module?

**Answer:** Manifest, tests, help, semantic version and signed or trusted source

Gallery quality.

## Practical coding questions

### A log parser with reports and an alert

Implement ConvertFrom-LogLine (named-group regex, typed properties, skip bad lines with Write-Verbose), then use the pipeline to report counts by level, the slowest requests, the server error rate, per-service average latency and an alert flag.

Starter code:

```
$log = @"
2026-03-01 10:00:01 INFO  web  GET /home 200 35ms
2026-03-01 10:00:02 ERROR api  POST /pay 500 910ms
2026-03-01 10:00:03 INFO  web  GET /cart 200 48ms
2026-03-01 10:01:10 WARN  api  GET /items 200 700ms
2026-03-01 10:01:11 ERROR api  POST /pay 500 880ms
2026-03-01 10:02:00 INFO  web  GET /home 404 12ms
bad line that should be skipped
"@ -split "`n"

function ConvertFrom-LogLine {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][string]$Line)
    process {
        # match "<date> <time> <LEVEL> <service> <METHOD> <path> <status> <ms>ms" with named groups;
        # emit [pscustomobject] with Time ([datetime]), Level, Service, Path, Status ([int]), Ms ([int]);
        # for lines that do not match, Write-Verbose and emit nothing
    }
}

$entries = $log | ConvertFrom-LogLine
"parsed: $($entries.Count) of $($log.Count) lines"
"by level:"
$entries | Group-Object Level | Sort-Object Name | ForEach-Object { "  {0,-5} {1}" -f $_.Name, $_.Count }
"slowest:"
$entries | Sort-Object Ms -Descending | Select-Object -First 2 | ForEach-Object { "  {0} {1} {2}ms" -f $_.Service, $_.Path, $_.Ms }
$errRate = [math]::Round(100 * @($entries | Where-Object Status -ge 500).Count / $entries.Count, 1)
"server error rate: $errRate%"
$avg = $entries | Group-Object Service | ForEach-Object { [pscustomobject]@{ Service = $_.Name; AvgMs = [math]::Round(($_.Group | Measure-Object Ms -Average).Average, 1) } }
($avg | Sort-Object Service | ForEach-Object { "$($_.Service)=$($_.AvgMs)" }) -join "; "
$alert = $errRate -gt 10
"ALERT: $alert"
($entries | Select-Object Service, Path, Status | ConvertTo-Json -Compress -Depth 2).Length -gt 0
```

Expected output:

```text
parsed: 6 of 7 lines
by level:
  ERROR 2
  INFO  3
  WARN  1
slowest:
  api /pay 910ms
  api /pay 880ms
server error rate: 33.3%
api=830; web=31.7
ALERT: True
True
```

??? success "Show a solution"

    ```
    $log = @"
    2026-03-01 10:00:01 INFO  web  GET /home 200 35ms
    2026-03-01 10:00:02 ERROR api  POST /pay 500 910ms
    2026-03-01 10:00:03 INFO  web  GET /cart 200 48ms
    2026-03-01 10:01:10 WARN  api  GET /items 200 700ms
    2026-03-01 10:01:11 ERROR api  POST /pay 500 880ms
    2026-03-01 10:02:00 INFO  web  GET /home 404 12ms
    bad line that should be skipped
    "@ -split "`n"

    function ConvertFrom-LogLine {
        [CmdletBinding()]
        param([Parameter(Mandatory, ValueFromPipeline)][string]$Line)
        process {
            if ($Line -match '^(?<ts>\S+ \S+) (?<level>[A-Z]+)\s+(?<svc>\w+)\s+(?<method>[A-Z]+) (?<path>\S+) (?<status>\d{3}) (?<ms>\d+)ms$') {
                [pscustomobject]@{
                    Time = [datetime]$Matches.ts; Level = $Matches.level; Service = $Matches.svc
                    Path = $Matches.path; Status = [int]$Matches.status; Ms = [int]$Matches.ms
                }
            } else { Write-Verbose "skipped: $Line" }
        }
    }

    $entries = $log | ConvertFrom-LogLine
    "parsed: $($entries.Count) of $($log.Count) lines"
    "by level:"
    $entries | Group-Object Level | Sort-Object Name | ForEach-Object { "  {0,-5} {1}" -f $_.Name, $_.Count }
    "slowest:"
    $entries | Sort-Object Ms -Descending | Select-Object -First 2 | ForEach-Object { "  {0} {1} {2}ms" -f $_.Service, $_.Path, $_.Ms }
    $errRate = [math]::Round(100 * @($entries | Where-Object Status -ge 500).Count / $entries.Count, 1)
    "server error rate: $errRate%"
    $avg = $entries | Group-Object Service | ForEach-Object { [pscustomobject]@{ Service = $_.Name; AvgMs = [math]::Round(($_.Group | Measure-Object Ms -Average).Average, 1) } }
    ($avg | Sort-Object Service | ForEach-Object { "$($_.Service)=$($_.AvgMs)" }) -join "; "
    $alert = $errRate -gt 10
    "ALERT: $alert"
    ($entries | Select-Object Service, Path, Status | ConvertTo-Json -Compress -Depth 2).Length -gt 0
    ```

    The function emits objects from process{} so the rest is ordinary pipeline work: Group-Object, Sort-Object, Measure-Object. Verified by running the solution with PowerShell 7.

### A validating pipeline function and retry helper

Implement Test-Password (validation attributes, a List of problems, masked output) and Invoke-WithRetry (retry with attempt numbers, rethrow after the last attempt). Run both against the sample inputs.

Starter code:

```
function Test-Password {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][AllowEmptyString()][string]$Password,
          [ValidateRange(4,64)][int]$MinLength = 8)
    process {
        # collect problems: "too short", "no uppercase" (case-sensitive test), "no digit", "no symbol" (non-word, non-space char)
        # emit [pscustomobject] Password (masked with '*' of the same length), Valid (bool), Problems (comma-separated)
    }
}

function Invoke-WithRetry {
    param([scriptblock]$Action, [int]$Max = 3)
    # run $Action with the attempt number; on failure write "attempt N failed: <message>" and retry;
    # rethrow after the last attempt; return the action's result on success
}

'Passw0rd!', 'short1!', 'alllowercase1!', 'NoDigits!!', '' | Test-Password | ForEach-Object { "{0,-14} valid={1,-5} {2}" -f $_.Password, $_.Valid, $_.Problems }
try { Test-Password -Password 'x' -MinLength 2 } catch { "validation: rejected" }
$r = Invoke-WithRetry { param($n) if ($n -lt 3) { throw "flaky $n" } else { "ok on attempt $n" } }
$r
try { Invoke-WithRetry -Max 2 { throw "always" } } catch { "gave up: $($_.Exception.Message)" }
```

Expected output:

```text
*********      valid=True
*******        valid=False too short, no uppercase
************** valid=False no uppercase
**********     valid=False no digit
               valid=False too short, no uppercase, no digit, no symbol
validation: rejected
attempt 1 failed: flaky 1
attempt 2 failed: flaky 2
ok on attempt 3
attempt 1 failed: always
attempt 2 failed: always
gave up: always
```

??? success "Show a solution"

    ```
    function Test-Password {
        [CmdletBinding()]
        param([Parameter(Mandatory, ValueFromPipeline)][AllowEmptyString()][string]$Password,
              [ValidateRange(4,64)][int]$MinLength = 8)
        process {
            $problems = [System.Collections.Generic.List[string]]::new()
            if ($Password.Length -lt $MinLength) { $problems.Add("too short") }
            if ($Password -cnotmatch '[A-Z]') { $problems.Add("no uppercase") }
            if ($Password -notmatch '\d') { $problems.Add("no digit") }
            if ($Password -notmatch '[^\w\s]') { $problems.Add("no symbol") }
            [pscustomobject]@{ Password = ('*' * $Password.Length); Valid = ($problems.Count -eq 0); Problems = ($problems -join ', ') }
        }
    }

    function Invoke-WithRetry {
        param([scriptblock]$Action, [int]$Max = 3)
        for ($i = 1; $i -le $Max; $i++) {
            try { return & $Action $i }
            catch { Write-Output "attempt $i failed: $($_.Exception.Message)"; if ($i -eq $Max) { throw } }
        }
    }

    'Passw0rd!', 'short1!', 'alllowercase1!', 'NoDigits!!', '' | Test-Password | ForEach-Object { "{0,-14} valid={1,-5} {2}" -f $_.Password, $_.Valid, $_.Problems }
    try { Test-Password -Password 'x' -MinLength 2 } catch { "validation: rejected" }
    $r = Invoke-WithRetry { param($n) if ($n -lt 3) { throw "flaky $n" } else { "ok on attempt $n" } }
    $r
    try { Invoke-WithRetry -Max 2 { throw "always" } } catch { "gave up: $($_.Exception.Message)" }
    ```

    Parameter validation rejects bad arguments before the body runs; process{} handles each piped value; try/catch with a loop gives retries and rethrows the final failure. Verified by running the solution with PowerShell 7.

---

More free practice: the [PowerShell Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
