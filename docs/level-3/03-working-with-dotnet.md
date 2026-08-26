# 03 · Working with .NET from PowerShell

Every PowerShell value is a .NET object under the hood — `"hello"` is a
`System.String`, `5` is a `System.Int32`, and `Get-Process` returns
`System.Diagnostics.Process` objects. That means the entire .NET class
library is available directly from your scripts, without a cmdlet
wrapper needing to exist for it first.

## Static members: `[Type]::Member`

```powershell
[Math]::Pow(2, 10)
[Math]::Round(3.14159, 2)
[DateTime]::Now.ToString("yyyy-MM-dd")
[string]::Join("-", @("a", "b", "c"))
```

```text
1024
3.14
2026-08-26
a-b-c
```

Square brackets around a type name (`[Math]`, `[DateTime]`, `[string]`)
access it directly; `::` calls a **static** member — one that belongs to
the type itself, not to any particular instance. There's no `New-Math`
cmdlet and there doesn't need to be one.

## Instantiating .NET objects with `::new()`

```powershell
$sw = [System.Diagnostics.Stopwatch]::StartNew()
Start-Sleep -Milliseconds 50
$sw.Stop()
"Elapsed: $($sw.ElapsedMilliseconds)ms"
```

```text
Elapsed: 53ms
```

`::new(...)` calls a constructor — the modern equivalent of `New-Object`,
and preferred because it supports overload resolution and IntelliSense
more reliably. Once created, `.Property` and `.Method()` work exactly
like they do on any PowerShell object, because it *is* one.

## `StringBuilder`: when `+=` on strings gets expensive

```powershell
$sb = [System.Text.StringBuilder]::new()
1..5 | ForEach-Object { [void]$sb.AppendLine("Line $_") }
$sb.ToString()
```

```text
Line 1
Line 2
Line 3
Line 4
Line 5
```

Each `$result += "text"` on a regular string silently allocates a whole
new string (strings are immutable in .NET) — fine for a handful of
appends, but O(n²) work if you're building a large report in a loop.
`StringBuilder` mutates an internal buffer instead. `[void]` in front of
`.AppendLine(...)` suppresses its return value — `AppendLine` returns the
`StringBuilder` itself (to allow chaining in C#), and without `[void]`
that return value would print to the pipeline on every call.

## `[regex]` directly, for named groups and reuse

```powershell
$regex = [regex]::new('(\d{3})-(\d{4})')
$m = $regex.Match("Call 555-1234 now")
"Match: $($m.Value), area: $($m.Groups[1].Value), rest: $($m.Groups[2].Value)"
```

```text
Match: 555-1234, area: 555, rest: 1234
```

`-match` (covered in Level 2) is fine for a one-off check, but
instantiating `[regex]` once and reusing it avoids recompiling the
pattern on every call in a hot loop — worth it once you're matching
inside a loop over thousands of lines.

## Generic collections: `List<T>` and `Dictionary<K,V>`

```powershell
$list = [System.Collections.Generic.List[int]]::new()
$list.Add(1); $list.Add(2); $list.Add(3)
$list.Contains(2)     # True
$list.Remove(2)       # True
$list -join ","       # 1,3
```

```powershell
$dict = [System.Collections.Generic.Dictionary[string,int]]::new()
$dict["a"] = 1
$dict["b"] = 2
foreach ($kvp in $dict.GetEnumerator()) { "$($kvp.Key)=$($kvp.Value)" }
```

```text
a=1
b=2
```

PowerShell arrays (`@()`) and hashtables (`@{}`) cover most needs, but
generic collections are strongly typed (`List[int]` rejects a string),
faster for large datasets since there's no boxing/type-checking overhead
per operation, and expose methods (`.BinarySearch()`, `.Sort()`,
`.TryGetValue()`) arrays don't have.

## The trap: a void-returning method looks like it "worked" either way

```powershell
$list = [System.Collections.Generic.List[string]]::new()
$result = $list.Add("x")   # Add returns void
"Result of Add(): [$result]"
```

```text
Result of Add(): []
```

`$result` is empty, not `$null` and not an error — `Add()` genuinely has
no return value in .NET, and PowerShell doesn't complain about capturing
nothing. The trap is assuming a captured variable proves the operation
succeeded or gives you useful information; check the .NET documentation
for a method's actual return type before relying on what you captured
from it. Some methods return `$true`/`$false` for success (`Remove`),
some return the item itself (`Dictionary.Add` returns void too, but
indexer assignment doesn't), and some return void — there's no single
rule, you have to know the specific method.

## Wrapping .NET behavior in a PowerShell `class`

You can't reopen or subclass most .NET types from PowerShell, but you can
compose one inside your own class:

```powershell
class Temperature {
    [double]$Celsius
    Temperature([double]$c) { $this.Celsius = $c }
    [double] ToFahrenheit() { return $this.Celsius * 9 / 5 + 32 }
    [string] ToString() { return "$($this.Celsius)C" }
}

$t = [Temperature]::new(100)
"$t is $($t.ToFahrenheit())F"
```

```text
100C is 212F
```

Overriding `ToString()` controls how the object renders when interpolated
into a string (as above) or piped to `Write-Output` — without it, you'd
see the default `Temperature` type name instead.

## Catching .NET exceptions with type-specific `catch`

```powershell
try {
    [int]"abc"
} catch {
    "Caught: $($_.Exception.GetType().Name): $($_.Exception.Message)"
}
```

```text
Caught: RuntimeException: Cannot convert value "abc" to type "System.Int32".
Error: "The input string 'abc' was not in a correct format."
```

A failed type conversion throws — wrap it, and inspect
`$_.Exception.GetType().Name` when you need to branch on *which* .NET
exception occurred (`FormatException`, `ArgumentOutOfRangeException`,
`IOException`) rather than treating every failure identically.

## Cheat sheet

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

## Exercise

Write a function `Measure-TextStats` that takes a block of text, uses
`[System.Text.StringBuilder]` internally to normalize it (lowercase, strip
punctuation via `[regex]::Replace`), then returns a `[pscustomobject]`
with word count, unique word count (using a
`[System.Collections.Generic.HashSet[string]]`), and the most common word.
Time it against a naive `+=` string-concatenation version on a
few-thousand-word input using `[System.Diagnostics.Stopwatch]` and confirm
the `StringBuilder` version is faster.
