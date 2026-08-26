# 06 · Performance at Scale

Level 3 covered performance for in-memory collections of a few tens of
thousands of items. At real scale — files with hundreds of thousands of
lines, repeated lookups against large datasets — different bottlenecks
dominate, and the fix is usually an algorithmic one (a better data
structure) rather than a syntactic one (a faster loop shape).

## Repeated lookups: hashtable vs. linear array search

```powershell
$n = 200000
$array = 1..$n
$lookup = @{}
1..$n | ForEach-Object { $lookup[$_] = $true }
$targets = 1..200 | ForEach-Object { Get-Random -Minimum 1 -Maximum $n }

$t1 = Measure-Command {
    foreach ($t in $targets) { $null = $array -contains $t }
}
"Array -contains x200 lookups: $($t1.TotalMilliseconds) ms"

$t2 = Measure-Command {
    foreach ($t in $targets) { $null = $lookup.ContainsKey($t) }
}
"Hashtable ContainsKey x200 lookups: $($t2.TotalMilliseconds) ms"
```

```text
Array -contains x200 lookups: 371.8268 ms
Hashtable ContainsKey x200 lookups: 4.9927 ms
```

**~74x** faster. `-contains` on an array scans every element until it
finds a match (or doesn't) — O(n) per lookup, so 200 lookups against
200,000 items means up to 40 million comparisons. A hashtable computes
where a key belongs directly (O(1) average) regardless of how many keys
it holds. The lesson generalizes past this exact example: any time a
script does the same kind of lookup repeatedly against a large,
mostly-static collection, building a hashtable (or
`System.Collections.Generic.HashSet[T]` when you only need
membership-checking, not a value) once up front pays for itself almost
immediately — the earlier single-lookup test at n=50,000 actually favored
the array slightly, because building the hashtable has fixed setup cost
that only amortizes once you do *enough* repeated lookups against it.

## Reading large files: three approaches compared

```powershell
1..500000 | Set-Content ./bignums.txt

$t1 = Measure-Command {
    $sum = 0
    Get-Content ./bignums.txt | ForEach-Object { $sum += [int]$_ }
}
"Get-Content | ForEach-Object (streams): $($t1.TotalMilliseconds) ms"

$t2 = Measure-Command {
    $lines = Get-Content ./bignums.txt -Raw
    $sum = 0
    foreach ($line in $lines -split "`n") {
        if ($line) { $sum += [int]$line }
    }
}
"Get-Content -Raw + split (loads whole file): $($t2.TotalMilliseconds) ms"

$t3 = Measure-Command {
    $sum = 0
    foreach ($line in [System.IO.File]::ReadLines("$PWD/bignums.txt")) {
        $sum += [int]$line
    }
}
".NET StreamReader via ReadLines: $($t3.TotalMilliseconds) ms"
```

```text
Get-Content | ForEach-Object (streams): 3537.9262 ms
Get-Content -Raw + split (loads whole file): 1064.6048 ms
.NET StreamReader via ReadLines: 855.3901 ms
```

The result here is worth sitting with, because it cuts against the usual
"streaming is always faster" instinct: `Get-Content` piped line-by-line
into `ForEach-Object` was the **slowest** of the three, at roughly 3.3-4x
the .NET version — `Get-Content`'s per-line pipeline overhead (each line
becomes a full pipeline object handoff) dominates for a large file, even
though it uses the least peak memory. `[System.IO.File]::ReadLines()`
(a .NET static method) skips PowerShell's pipeline machinery entirely
while still streaming line-by-line — it gets the streaming's memory
benefit *and* the speed of avoiding per-line pipeline overhead, which is
why it's fastest here. `-Raw` loads the whole file into one string
upfront (highest memory use, no streaming benefit at all), landing in the
middle.

The practical takeaway: for very large files where both memory and speed
matter, reach for `[System.IO.File]::ReadLines()` over plain
`Get-Content`, and reserve plain `Get-Content` for files small enough that
the difference doesn't matter (its readability and pipeline-friendliness
are real advantages when performance isn't the constraint).

## Lazy evaluation and short-circuiting: `Select-Object -First`

```powershell
$t3 = Measure-Command {
    $first = 1..10000000 | Select-Object -First 1
}
"Select-Object -First 1 on 10M range: $($t3.TotalMilliseconds) ms"
```

```text
Select-Object -First 1 on 10M range: 4.6485 ms
```

Under 5ms against a 10-million-element range — `Select-Object -First N`
stops pulling from the pipeline the moment it has enough items, rather
than materializing the whole 10 million first. This is the pipeline's
*actual* strength at scale: for anything you can express as "give me the
first N", "stop once you find one", or similar, keeping it in the
pipeline and using `-First`/`Where-Object` short-circuiting can beat
fully loading a collection into memory just to slice it afterward.

## The trap: assuming yesterday's benchmark still holds

The hashtable-vs-array and `Get-Content`-vs-`.NET` numbers above are
specific to this machine, this PowerShell version, and these input
sizes — the actual crossover points shift with hardware, .NET version,
and data shape. The habit that matters isn't memorizing these specific
numbers, it's reaching for `Measure-Command` (module 07, Level 3) on
*your* actual data before optimizing, and re-checking after a PowerShell
or .NET upgrade rather than assuming last year's optimization still holds.

## Cheat sheet

| Situation | Prefer |
|---|---|
| Repeated membership/lookup checks against a large, static set | hashtable or `HashSet[T]`, built once |
| One-off `-contains` check on a small array | plain array is fine, don't over-engineer |
| Reading a very large file, need speed and low memory | `[System.IO.File]::ReadLines()` |
| Reading a large file, readability matters more than raw speed | `Get-Content` (streams, pipeline-friendly) |
| Need the whole file as one string (regex across lines, etc.) | `Get-Content -Raw` (accept the memory cost) |
| Need only the first N results (or first match) | keep it in the pipeline, use `-First`/short-circuit `Where-Object` |
| Any performance claim | verify with `Measure-Command` on your actual data, don't assume |

## Exercise

Take a script that processes a 200,000-line CSV log file line-by-line
with `Get-Content`, checking each row's ID against a list of 5,000
"flagged" IDs using `-contains`. Rewrite it to use
`[System.IO.File]::ReadLines()` for reading and a `HashSet[string]` for
the flagged-ID lookup, and use `Measure-Command` to confirm the rewrite
is substantially faster on a realistically sized test file you generate
yourself.
