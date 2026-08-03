# 01 · Advanced Pipeline

Level 1 introduced `Where-Object`, `ForEach-Object`, `Sort-Object`, and
`Group-Object` as individual tools. This module goes one level deeper: how
the pipeline actually moves objects one at a time, why that matters for
performance and correctness, and the less obvious behaviors of filtering,
projecting, and looping that trip people up once scripts get bigger.

## The pipeline processes one object at a time

A pipeline like `Get-Process | Where-Object { $_.WorkingSet64 -gt 100MB } |
Sort-Object WorkingSet64` looks like three separate passes over the data,
but `Where-Object` doesn't wait for all processes to arrive before
filtering — each object flows through the whole chain individually before
the next one starts.

```powershell
1..5 | ForEach-Object {
    Write-Output "Processing $_"
    $_ * 2
} | ForEach-Object {
    Write-Output "  received $_"
}
```

```text
Processing 1
  received 2
Processing 2
  received 4
Processing 3
  received 6
Processing 4
  received 8
Processing 5
  received 10
```

Notice the interleaving — object `1` runs through *both* stages before
object `2` is even produced. `Sort-Object` and `Group-Object` are
exceptions: they must buffer every object first (you can't know an item is
"the smallest" until you've seen them all), which is why a big pipeline
with a `Sort-Object` in the middle uses more memory than one that only
filters and transforms.

## Why this beats "load everything, then loop"

```powershell
# Streaming: memory stays flat no matter how many log lines exist
Get-Content huge.log | Where-Object { $_ -match "ERROR" } | Select-Object -First 10

# Non-streaming: the whole file is an array in memory before anything happens
$lines = Get-Content huge.log
$lines | Where-Object { $_ -match "ERROR" } | Select-Object -First 10
```

Both give the same result here, but the first version can stop reading the
file as soon as `Select-Object -First 10` has what it needs — PowerShell
propagates a "stop asking for more" signal back up the pipeline. The second
version reads the entire file into `$lines` regardless of how much of it you
actually use.

## `$_` and `$PSItem` are the same thing

```powershell
Get-Process | Where-Object { $_.ProcessName -eq $PSItem.ProcessName }
```

`$PSItem` was added as a more readable alias for `$_` — they always refer to
the same current pipeline object. Pick one convention and stay consistent;
this course uses `$_` because it's what you'll see in the overwhelming
majority of real-world scripts and documentation.

## The trap: `$_` doesn't survive into a nested scriptblock

```powershell
# This does NOT do what it looks like it does
1..3 | ForEach-Object {
    $current = $_
    1..2 | ForEach-Object {
        # Inside here, $_ is the INNER loop's value, not $current's
        Write-Output "outer=$current inner=$_"
    }
}
```

```text
outer=1 inner=1
outer=1 inner=2
outer=2 inner=1
outer=2 inner=2
outer=3 inner=1
outer=3 inner=2
```

This actually works correctly *because* `$current` was captured into its own
variable — the trap is forgetting to do that and using `$_` directly inside
a nested pipeline, where it silently refers to whichever loop is innermost.
Always capture `$_` into a named variable before entering a nested
pipeline or calling another function that also uses `$_`.

## `begin` / `process` / `end`: what `ForEach-Object` is built from

Every pipeline-aware function has three optional blocks. `process` is the
one that runs per object (the "for each" part); `begin` runs once before
the first object arrives; `end` runs once after the last one.

```powershell
function Measure-Lines {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline = $true)]
        [string]$Line
    )

    begin {
        Write-Output "Starting..."
        $count = 0
        $totalChars = 0
    }
    process {
        $count++
        $totalChars += $Line.Length
    }
    end {
        Write-Output "Processed $count lines, $totalChars characters total"
    }
}

"first", "second line", "c" | Measure-Lines
```

```text
Starting...
Processed 3 lines, 15 characters total
```

Without `begin`/`process`/`end`, a function only has a single body that runs
**once**, receiving the entire piped collection as an array — it can't
process items as they stream in, and can't emit output until the whole
input has been collected. That distinction matters most on large or
never-ending input (like a live log tail): a `process`-block function
starts emitting results immediately, while a plain function waits for
everything first.

```powershell
# Without process: only ever sees the FULL array in $Line, once
function Measure-LinesWrong {
    param(
        [Parameter(ValueFromPipeline = $true)]
        [string[]]$Line
    )
    Write-Output "Got $($Line.Count) line(s) in one shot"
}

"first", "second line", "c" | Measure-LinesWrong
# Got 3 line(s) in one shot   <-- ran once, not three times
```

## `ValueFromPipeline` vs `ValueFromPipelineByPropertyName`

```powershell
function Get-DoubledValue {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline = $true)]
        [int]$Value,

        [Parameter(ValueFromPipelineByPropertyName = $true)]
        [string]$ProcessName
    )
    process {
        [pscustomobject]@{
            Value   = $Value * 2
            Process = $ProcessName
        }
    }
}

5 | Get-DoubledValue
# Value Process
# ----- -------
#    10

Get-Process -Name pwsh | Select-Object -First 1 | Get-DoubledValue
# Value Process
# ----- -------
#     0 pwsh
```

- `ValueFromPipeline` binds the **whole piped object** to a parameter — only
  one parameter per function can use it directly for a scalar type.
- `ValueFromPipelineByPropertyName` binds a piped object's **property of the
  same name** to a parameter — this is how `Get-Process | Stop-Process`
  works even though `Stop-Process` takes an `-Id` or `-Name` parameter, not
  a raw process object: it matches by property name.

## Building a real begin/process/end function

```powershell
function ConvertTo-Fahrenheit {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline = $true)]
        [double]$Celsius
    )

    begin {
        $processedCount = 0
    }
    process {
        $processedCount++
        [pscustomobject]@{
            Celsius    = $Celsius
            Fahrenheit = [math]::Round(($Celsius * 9 / 5) + 32, 1)
        }
    }
    end {
        Write-Verbose "Converted $processedCount temperature(s)"
    }
}

0, 20, 37, 100 | ConvertTo-Fahrenheit
```

```text
Celsius Fahrenheit
------- ----------
      0         32
     20         68
     37       98.6
    100        212
```

## `Where-Object` and `ForEach-Object` in their fast, "-Property" form

```powershell
# Script block form (most flexible)
Get-Process | Where-Object { $_.WorkingSet64 -gt 200MB }

# Comparison-statement form (no script block, slightly faster, less flexible)
Get-Process | Where-Object WorkingSet64 -gt 200MB

# ForEach-Object also has a shortcut form for calling a single member
Get-Process | ForEach-Object -MemberName Kill      # equivalent to { $_.Kill() }
"3.14", "2.71" | ForEach-Object -MemberName Trim    # equivalent to { $_.Trim() }
```

The shortcut forms exist mainly for readability in short one-liners; once a
condition needs more than one comparison, drop back to a script block.

## Controlling flow inside the pipeline: `continue` vs `return`

```powershell
1..5 | ForEach-Object {
    if ($_ -eq 3) { return }   # skips just this iteration, like "continue" in a foreach loop
    Write-Output $_
}
```

```text
1
2
4
5
```

Inside a `ForEach-Object` script block, `return` only exits **that one
call** of the block — not the whole pipeline — because each object gets its
own invocation of the block. This surprises people coming from C-like
languages who expect `return` to exit the enclosing function.

## Cheat sheet

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

## Exercise

Write a function `Get-WordStats` with `[CmdletBinding()]` and a
`[Parameter(ValueFromPipeline = $true)]` string parameter called `Line`.
Using `begin`/`process`/`end`, have it count the total number of lines and
total number of words (split on whitespace) piped into it, and in `end`
emit a single `[pscustomobject]` with `LineCount` and `WordCount`
properties. Test it with:
`"the quick fox", "jumps over", "the lazy dog" | Get-WordStats`.
