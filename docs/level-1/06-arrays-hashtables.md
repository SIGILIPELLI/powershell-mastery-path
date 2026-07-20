# 06 · Arrays & Hashtables

## Creating arrays

```powershell
$fruits = "apple", "banana", "cherry"     # comma creates an array
$numbers = @(1, 2, 3, 4, 5)                # explicit array syntax
$empty = @()                                # empty array

$fruits[0]      # apple
$fruits[-1]     # cherry   <- negative indices count from the end
```

```powershell
# arrays are typed as System.Object[] by default — mixed types are allowed
$mixed = 1, "two", 3.0, $true
$mixed.GetType().Name   # Object[]

# strongly-typed arrays constrain every element
[int[]]$scores = 90, 85, 77
```

## Common array operations

```powershell
$numbers = 1, 2, 3, 4, 5

$numbers.Count            # 5
$numbers.Length            # 5   <- same as .Count for arrays

$numbers + 6                # returns a NEW array: 1 2 3 4 5 6 (arrays are fixed-size)
$numbers[1..3]              # slice: 2 3 4
$numbers[0,2,4]             # index list: 1 3 5
```

```powershell
# arrays are fixed-size; += actually creates a new array each time (slow for big loops)
$list = @()
$list += "a"
$list += "b"
# fine for small scripts; use a generic List[T] or ArrayList for large/growing collections
```

## Growable collections: ArrayList and List[T]

```powershell
$list = [System.Collections.Generic.List[string]]::new()
$list.Add("a")
$list.Add("b")
$list.Remove("a")
$list          # b
```

```powershell
# a strongly-typed generic list is the recommended growable collection
$scores = [System.Collections.Generic.List[int]]::new()
$scores.AddRange(@(10, 20, 30))
$scores.Add(40)
$scores.Count   # 4
```

## Iterating arrays

```powershell
$fruits = "apple", "banana", "cherry"

foreach ($fruit in $fruits) {
    Write-Output "I like $fruit"
}

$fruits | ForEach-Object { Write-Output "Pipeline: $_" }
```

## Hashtables

Hashtables are PowerShell's key-value dictionary — similar to a Python dict
or a Bash associative array, but a first-class type usable everywhere.

```powershell
$capitals = @{
    France  = "Paris"
    Japan   = "Tokyo"
    Germany = "Berlin"
}

$capitals["Japan"]     # Tokyo
$capitals.Japan         # Tokyo   <- dot notation also works for simple keys
```

```powershell
# adding, updating, removing
$capitals["Italy"] = "Rome"      # add
$capitals["Japan"] = "Kyoto"     # update (overwrite)
$capitals.Remove("Germany")       # remove

$capitals.ContainsKey("France")   # True
$capitals.Keys                    # France, Japan, Italy
$capitals.Values                  # Paris, Kyoto, Rome
```

## Iterating a hashtable

```powershell
$capitals = @{ France = "Paris"; Japan = "Tokyo"; Germany = "Berlin" }

foreach ($key in $capitals.Keys) {
    Write-Output "$key -> $($capitals[$key])"
}

# or, using GetEnumerator() to get key/value pairs directly
$capitals.GetEnumerator() | ForEach-Object {
    Write-Output "$($_.Key) -> $($_.Value)"
}
```

By default, `@{}` is an unordered `Hashtable` — key order isn't guaranteed.
Use `[ordered]@{}` when insertion order matters:

```powershell
$ordered = [ordered]@{ First = 1; Second = 2; Third = 3 }
$ordered.Keys    # First, Second, Third   <- guaranteed insertion order
```

## A practical example: word frequency count

```powershell
$text = "the quick brown fox jumps over the lazy dog the fox runs"
$counts = @{}

foreach ($word in $text -split " ") {
    if ($counts.ContainsKey($word)) {
        $counts[$word]++
    } else {
        $counts[$word] = 1
    }
}

$counts.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 3
```

```text
Name  Value
----  -----
the       3
fox       2
quick     1
```

## Cheat sheet

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

## Exercise

Write `inventory.ps1` that builds a hashtable mapping item names to
quantities (e.g. `apples = 12`, `bananas = 7`, `oranges = 20`), prints each
item and its quantity sorted by quantity descending using
`GetEnumerator() | Sort-Object`, and prints the total quantity across all
items using `Measure-Object -Sum` on the hashtable's values.
