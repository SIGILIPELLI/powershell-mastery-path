# 07 · Performance Optimization

Scripts that process a few dozen items don't need to think about
performance. Scripts that process tens of thousands do — and PowerShell
has a handful of patterns that look almost identical on the surface but
differ by 10-20x in real runtime. This module measures them directly
rather than asserting it.

## The classic trap: `$array += $item` in a loop

```powershell
$n = 20000

$t1 = Measure-Command {
    $arr = @()
    for ($i = 0; $i -lt $n; $i++) { $arr += $i }
}
"Array += : $($t1.TotalMilliseconds) ms"

$t2 = Measure-Command {
    $list = [System.Collections.Generic.List[int]]::new()
    for ($i = 0; $i -lt $n; $i++) { $list.Add($i) }
}
"List<T>.Add : $($t2.TotalMilliseconds) ms"

$t3 = Measure-Command {
    $result = for ($i = 0; $i -lt $n; $i++) { $i }
}
"for-loop output capture : $($t3.TotalMilliseconds) ms"
```

```text
Array += : 672.9626 ms
List<T>.Add : 36.2933 ms
for-loop output capture : 28.1635 ms
```

Nearly **20x** slower for `+=`. PowerShell arrays (`@()`) are fixed-size
.NET arrays under the hood — `+=` doesn't append in place, it silently
allocates an entirely new array one element larger and copies everything
over, every single time. At 20,000 iterations that's 20,000 full-array
copies. `List[T].Add()` and just letting a loop emit values to be
captured by `$result =` both avoid that entirely — prefer whichever fits
the code better; the list is best when you need random access or removal
mid-loop, and letting the loop's own output accumulate is simplest when
you don't.

## Pipeline vs `foreach` vs array methods

```powershell
$data = 1..100000

$t1 = Measure-Command {
    $data | Where-Object { $_ % 2 -eq 0 } | ForEach-Object { $_ * 2 } | Out-Null
}
"Pipeline (Where-Object|ForEach-Object): $($t1.TotalMilliseconds) ms"

$t2 = Measure-Command {
    foreach ($x in $data) {
        if ($x % 2 -eq 0) { $x * 2 | Out-Null }
    }
}
"foreach loop: $($t2.TotalMilliseconds) ms"

$t3 = Measure-Command {
    $data.Where({ $_ % 2 -eq 0 }).ForEach({ $_ * 2 }) | Out-Null
}
"Where()/ForEach() array methods: $($t3.TotalMilliseconds) ms"
```

```text
Pipeline (Where-Object|ForEach-Object): 600.369 ms
foreach loop: 194.5552 ms
Where()/ForEach() array methods: 156.2134 ms
```

The `|` pipeline is roughly **3-4x slower** here than either alternative.
Every object crossing a `|` goes through PowerShell's pipeline machinery —
binding parameters, invoking a scriptblock per item — which has real
overhead per element. `foreach` and the `.Where()`/`.ForEach()` array
intrinsic methods skip that machinery for in-memory collections. This
doesn't mean "never use the pipeline" — its streaming behavior (each
object flows through immediately rather than waiting for the whole
collection) is exactly what you want for large files or live data you
don't want fully buffered in memory. It means: for CPU-bound
transformations over a collection *already sitting in memory*, `foreach`
or `.Where()/.ForEach()` are the faster default.

## The trap: expensive formatting/display cmdlets inside a loop

```powershell
$objs = 1..2000 | ForEach-Object { [pscustomobject]@{ Id = $_; Name = "item$_" } }

$t4 = Measure-Command {
    foreach ($o in $objs) { $o | Format-Table | Out-Null }
}
"Format-Table INSIDE loop (bad): $($t4.TotalMilliseconds) ms"

$t5 = Measure-Command {
    $objs | Format-Table | Out-Null
}
"Format-Table ONCE at end (good): $($t5.TotalMilliseconds) ms"
```

```text
Format-Table INSIDE loop (bad): 291.8978 ms
Format-Table ONCE at end (good): 22.6889 ms
```

Over **12x** slower calling `Format-Table` once per item versus once for
the whole collection — each call sets up formatting state (column widths,
headers) from scratch. This generalizes: any cmdlet meant for *final
display* (`Format-Table`, `Format-List`, `Out-GridView`, `Write-Host`
with heavy formatting) belongs after a loop finishes collecting data, not
inside the loop doing per-item work.

## Measuring, not guessing

```powershell
Measure-Command { <script block> }
```

returns a `TimeSpan` — `.TotalMilliseconds`, `.TotalSeconds`, etc. It's
the right tool for **comparing two approaches**, but note it suppresses
the script block's normal output (it returns only the timing), so add
`| Out-Null` inside if you don't want return values leaking into the
timing itself, and never trust a single run — background system load
varies; run each variant 3+ times and compare typical values, not
one-off numbers.

For finding *where* time actually goes in a larger script rather than
comparing two known alternatives, `Trace-Command` and the cross-platform
profiler modules (e.g. `PSProfiler` from the gallery) go deeper than
`Measure-Command` can alone.

## Cheat sheet

| Pattern | Verdict |
|---|---|
| `$array += $item` in a loop | avoid — O(n²), full copy every iteration |
| `[List[T]]::new()` + `.Add()` | fast, in-place growth |
| loop output captured by `$result = for (...) { }` | fast, often the simplest fix |
| `\|` pipeline over an in-memory collection | slower per-item, but streams (low memory) |
| `foreach` / `.Where()` / `.ForEach()` over in-memory data | faster for pure CPU-bound work |
| `Format-*`, `Out-GridView` inside a loop | avoid — format once, after the loop |
| `Measure-Command { }` | compare two implementations directly, run several times |

## Exercise

Take a script that builds a report by looping over 50,000 numbers,
filtering to multiples of 7, doubling them, and appending each result to
an array with `+=`, then calling `Format-Table` inside the same loop.
Rewrite it using a `foreach` loop, a `[List[int]]` (or loop-output
capture), and a single `Format-Table` call at the end. Time both versions
with `Measure-Command` and confirm the rewrite is meaningfully faster.
