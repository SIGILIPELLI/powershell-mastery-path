# 05 · Working with Objects & the Pipeline

This is the single biggest thing that sets PowerShell apart from Bash and
most other shells. In Bash, pipes (`|`) pass **plain text** between commands
— every program has to parse that text back apart with tools like `awk` or
`grep`. In PowerShell, pipes pass **live .NET objects**, with named
properties and methods intact, from one cmdlet straight into the next.

## A first look

```powershell
Get-Process | Select-Object -First 3
```

```text
NPM(K)    PM(M)      WS(M)     CPU(s)      Id  SI ProcessName
------    -----      -----     ------      --  -- -----------
    25    45.30      52.11       1.20    1234   1 chrome
    18    12.44      15.02       0.34    5678   1 Finder
    32    88.90      95.40       4.11    9012   1 pwsh
```

`Get-Process` doesn't print text — it returns a collection of
`System.Diagnostics.Process` **objects**. What you see is just the default
*formatted view* of those objects.

## Inspecting an object

```powershell
$proc = Get-Process | Select-Object -First 1

$proc.GetType().FullName     # System.Diagnostics.Process
$proc | Get-Member            # lists every property and method available
```

```text
   TypeName: System.Diagnostics.Process

Name                       MemberType     Definition
----                       ----------     ----------
Kill                       Method         void Kill()
Refresh                    Method         void Refresh()
Id                         Property       int Id {get;}
ProcessName                Property       string ProcessName {get;}
StartTime                  Property       datetime StartTime {get;}
WorkingSet64               Property       long WorkingSet64 {get;}
```

`Get-Member` is the single most useful discovery command in PowerShell —
when you don't know what a piped object can do, pipe it to `Get-Member`.

## Selecting properties

```powershell
Get-Process | Select-Object -Property ProcessName, Id, WorkingSet64 -First 3
```

```powershell
# dotted access on a single object
$proc = Get-Process -Name pwsh | Select-Object -First 1
Write-Output $proc.ProcessName
Write-Output $proc.Id
```

## Filtering with Where-Object

```powershell
Get-Process | Where-Object { $_.WorkingSet64 -gt 100MB }
```

`$_` refers to "the current object in the pipeline" — you'll see it
constantly. `100MB` is a built-in unit literal (also `KB`, `GB`, `TB`).

```powershell
# simplified comparison-based syntax (no script block needed)
Get-Process | Where-Object WorkingSet64 -gt 100MB
```

## Sorting and grouping

```powershell
Get-Process | Sort-Object -Property WorkingSet64 -Descending | Select-Object -First 3

Get-Process | Group-Object -Property ProcessName | Sort-Object Count -Descending | Select-Object -First 3
```

```text
Count Name                      Group
----- ----                      -----
    4 chrome helper              {chrome, chrome, chrome, chrome}
    2 pwsh                       {pwsh, pwsh}
    1 Finder                     {Finder}
```

## Transforming with ForEach-Object

```powershell
1..5 | ForEach-Object { $_ * $_ }
# 1
# 4
# 9
# 16
# 25
```

```powershell
Get-Process | ForEach-Object {
    "{0} is using {1:N1} MB" -f $_.ProcessName, ($_.WorkingSet64 / 1MB)
} | Select-Object -First 3
```

## Chaining it all together

This is the real power of the pipeline: build up a data-processing chain
one filtered/sorted/shaped step at a time.

```powershell
Get-Process |
    Where-Object { $_.WorkingSet64 -gt 50MB } |
    Sort-Object WorkingSet64 -Descending |
    Select-Object -First 5 -Property ProcessName, @{Name="MemoryMB"; Expression={ [math]::Round($_.WorkingSet64 / 1MB, 1) }} |
    Format-Table -AutoSize
```

The `@{Name=...; Expression={...}}` hashtable defines a **calculated
property** — you can reshape data mid-pipeline without writing a separate
loop.

## Measuring

```powershell
Get-Process | Measure-Object -Property WorkingSet64 -Sum -Average -Maximum
```

```text
Count    : 210
Average  : 45832112.4
Sum      : 9624743604
Maximum  : 892034000
Minimum  :
Property : WorkingSet64
```

## Exporting and converting

```powershell
Get-Process | Select-Object ProcessName, Id | Export-Csv -Path processes.csv -NoTypeInformation
Get-Process | Select-Object ProcessName, Id | ConvertTo-Json | Out-File processes.json
```

Because the pipeline carries real objects, exporting to CSV/JSON "just
works" — no manual formatting needed, because each object already knows its
own property names and types.

## Cheat sheet

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

## How It Actually Works

PowerShell's pipeline is not a byte stream between processes the way a
Unix pipe is — it's a sequence of live .NET object references passed
one-at-a-time between cmdlets running on the *same thread* inside the
*same runspace*, with no serialization in between (serialization only
happens crossing a remoting boundary). Every value in the pipeline is
wrapped in a `PSObject`, which carries two layers: the underlying .NET
object (`BaseObject`) and an **adapted member set** — properties, script
methods, and note properties attached by `Update-TypeData`,
`ETS` (Extended Type System) rules in `*.types.ps1xml`, or ad hoc via
`Add-Member`. This is why `Get-Process | Select-Object Name, WS` can show
a `WS` property that isn't a real field on `System.Diagnostics.Process` —
it's an ETS alias property defined in PowerShell's own type data.

Cmdlet chaining (`Get-Process | Where-Object ... | Sort-Object ...`)
executes with **streaming, per-object semantics**, not batch-then-forward:
each cmdlet in the pipe is instantiated once, and the pipeline processor
calls the first cmdlet's `ProcessRecord` once, then immediately calls the
next cmdlet's `ProcessRecord` with that single object, all the way down
the chain, before returning for the next object. This is why a badly
ordered pipeline (`Sort-Object` before a slow filter) is slower — sorting
requires buffering the *entire* stream (it must see every object before
it can emit the first one), which breaks the streaming model and forces a
synchronous barrier, unlike a pure filter like `Where-Object` that can
pass objects through one at a time.

`Where-Object` and `ForEach-Object` predicates run as compiled
`ScriptBlock` delegates invoked once per pipeline object with `$_`
(`$PSItem`) bound in a fresh child scope — the "simplified syntax" forms
(`Where-Object Status -eq 'Running'`) bypass script-block compilation
entirely and instead build a comparison directly against the named
property via reflection/ETS lookup, which is marginally faster and also
why that syntax can't express compound conditions the way a script block
predicate can.

## Exercise

Write `top-processes.ps1` that gets all running processes, filters to those
using more than 20MB of working set, sorts by memory descending, selects the
top 5 with a calculated `MemoryMB` property (rounded to 1 decimal), and
exports the result to `top-processes.csv` using `Export-Csv`.
