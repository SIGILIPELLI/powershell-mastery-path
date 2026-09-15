---
description: "Functions — PowerShell function names conventionally follow Verb-Noun casing (e.g. Get-Process, New-Item) — a singular noun, and a verb from PowerShell's…"
---

# 04 · Functions

## Basic function syntax

```powershell
function Get-Greeting {
    Write-Output "Hello there!"
}

Get-Greeting
# Hello there!
```

PowerShell function names conventionally follow **Verb-Noun** casing (e.g.
`Get-Process`, `New-Item`) — a singular noun, and a verb from PowerShell's
approved-verb list. We'll cover approved verbs properly in Level 2; for now,
just follow the pattern.

## Parameters

```powershell
function Get-Greeting {
    param(
        [string]$Name
    )
    Write-Output "Hello, $Name!"
}

Get-Greeting -Name "Ada"
# Hello, Ada!

Get-Greeting "Ada"        # positional argument also works
```

```powershell
# multiple parameters, with default values
function New-Rectangle {
    param(
        [double]$Width = 1,
        [double]$Height = 1
    )
    $area = $Width * $Height
    Write-Output "Area: $area"
}

New-Rectangle -Width 4 -Height 5   # Area: 20
New-Rectangle                       # Area: 1   <- uses defaults
```

## Mandatory parameters and validation

```powershell
function New-User {
    param(
        [Parameter(Mandatory)]
        [string]$Username,

        [ValidateRange(0, 150)]
        [int]$Age = 18
    )
    Write-Output "Creating user '$Username', age $Age"
}

New-User -Username "ada"          # Creating user 'ada', age 18
New-User                           # prompts interactively for -Username since it's mandatory
New-User -Username "ada" -Age 200  # throws: Age must be between 0 and 150
```

`Mandatory` makes PowerShell prompt the user for the value if it's omitted,
rather than failing immediately — handy interactively, but usually you want
scripts to fail fast, which is exactly what happens when called
non-interactively without the parameter.

## Return values

```powershell
function Get-Square {
    param([int]$Number)
    return $Number * $Number
}

$result = Get-Square -Number 6
Write-Output $result   # 36
```

Any output that isn't captured, redirected, or assigned "falls through" and
becomes part of the function's return value — `return` is optional and
mostly used for early exits, not required for producing output.

```powershell
function Get-EvenSquares {
    param([int[]]$Numbers)
    foreach ($n in $Numbers) {
        if ($n % 2 -eq 0) {
            $n * $n     # no 'return' needed — this becomes output
        }
    }
}

Get-EvenSquares -Numbers 1,2,3,4,5,6
# 4
# 16
# 36
```

This is a common beginner trap: any *uncaptured* expression's value becomes
part of the output stream, even things like the result of `$i++` in some
contexts. Be deliberate about what you leave "bare" in a function body —
assign to `$null` or use `[void]` to suppress an unwanted value.

```powershell
function Add-ToList {
    param([System.Collections.ArrayList]$List, $Item)
    [void]$List.Add($Item)   # suppress .Add()'s return value (the new count)
}
```

## Advanced functions: CmdletBinding and pipeline input

Adding `[CmdletBinding()]` turns an ordinary function into an "advanced
function" that behaves like a real cmdlet — gaining common parameters like
`-Verbose` and `-ErrorAction` for free.

```powershell
function Get-Doubled {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline)]
        [int]$Number
    )
    process {
        $Number * 2
    }
}

1, 2, 3 | Get-Doubled
# 2
# 4
# 6
```

The `process { }` block runs once *per pipeline item* — essential for
functions meant to accept piped input (more in Level 2).

## Splatting: passing many parameters at once

```powershell
function New-Greeting {
    param([string]$Name, [string]$Greeting = "Hello")
    Write-Output "$Greeting, $Name!"
}

$params = @{
    Name     = "Ada"
    Greeting = "Hi"
}
New-Greeting @params    # Hi, Ada!   <- @ instead of $ "splats" the hashtable as named params
```

## Cheat sheet

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

## How It Actually Works

A PowerShell function is compiled into a `FunctionInfo`/`ScriptBlock`
pair stored in the session's function provider drive (`function:`) — which
is why you can literally do `Get-Item function:Get-Foo` or delete a
function with `Remove-Item function:Get-Foo`; functions live in the same
provider-drive abstraction as files.

Parameter binding is the part with real machinery behind it: when you call
a function, the engine's **parameter binder** does not just match
positional order. It runs a multi-pass algorithm — first binding all named
parameters, then binding remaining positional arguments to remaining
parameters in declared position order, then attempting to bind leftover
arguments via pipeline input (`ByValue`, then `ByPropertyName`) if the
function declares pipeline-aware parameters. This is the same binder
cmdlets use, so `param()` blocks with `[Parameter()]` attributes get
*identical* treatment to compiled cmdlet parameters — mandatory-parameter
prompting, `ValidateSet`/`ValidateRange` checks, and type coercion all run
during this binding phase, before your function body executes a single
line.

`[CmdletBinding()]` upgrades a function from a "simple function" to an
**advanced function**, which changes execution shape: the engine now calls
optional `Begin`/`Process`/`End` blocks the way it calls a compiled
cmdlet's `BeginProcessing`/`ProcessRecord`/`EndProcessing` — `Process` runs
once per pipeline input object, which is the actual mechanism behind
"pipeline input" rather than a documentation convention. Without
`CmdletBinding`, a function has no `Process` block semantics and pipeline
input just accumulates into `$input` for a single pass.

Return values are not built with a `return` keyword pushing one value onto
a stack — **anything not captured, suppressed, or redirected inside a
function body is written to the pipeline's output stream automatically**.
`return $x` is sugar for `Write-Output $x` followed by an early exit, which
is exactly why an unassigned `Get-ChildItem` call buried in the middle of
a function silently leaks extra objects into your function's output: the
engine has no concept of "the value the function produced" separate from
"everything the function's pipeline emitted."

## 🔀 See this in another language

- [Python — Functions & Scope](https://sigilipelli.github.io/python-mastery-path/level-1/04-functions-scope/)
- [C# — Methods & Functions](https://sigilipelli.github.io/csharp-mastery-path/level-1/04-methods-functions/)
- [Kotlin — Functions](https://sigilipelli.github.io/kotlin-mastery-path/level-1/04-functions/)

## Exercise

Write `Get-BMI.ps1` containing a function `Get-BMI` that takes mandatory
`-WeightKg` and `-HeightM` parameters (both `[double]`), computes
`weight / (height * height)`, and returns the value rounded to 1 decimal
place using `[math]::Round(...)`. Call it with a couple of test values and
print the results.
