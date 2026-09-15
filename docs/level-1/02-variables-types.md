---
description: "Variables & Types — PowerShell variables are dynamically typed by default but backed by real .NET types under the hood — every value you touch is an…"
---

# 02 · Variables & Types

PowerShell variables are dynamically typed by default but backed by real .NET
types under the hood — every value you touch is an *object* with a type,
methods, and properties, not just text.

## Declaring variables

```powershell
$name = "Ada"
$age = 36
$isActive = $true

Write-Output $name
Write-Output "$name is $age years old"   # string interpolation with double quotes
```

```text
Ada
Ada is 36 years old
```

Variable names start with `$`. Single quotes are literal (no interpolation);
double quotes interpolate `$variables` and `$(expressions)`.

```powershell
$count = 3
Write-Output 'Count is $count'          # literal: Count is $count
Write-Output "Count is $count"          # interpolated: Count is 3
Write-Output "Doubled: $($count * 2)"   # sub-expression: Doubled: 6
```

## Checking and converting types

```powershell
$age = 36
$age.GetType().Name        # Int32
"36".GetType().Name         # String
(3.14).GetType().Name       # Double
$true.GetType().Name        # Boolean
```

```powershell
# explicit type conversion (casting)
$text = "42"
$number = [int]$text
$number + 8                 # 50

# the opposite direction
[string]42                  # "42"
```

```powershell
# PowerShell also auto-converts in many contexts
$result = "5" + 3            # 8   <- numeric string + int => arithmetic
$result2 = 5 + "3"            # 8   <- same, order doesn't matter here
$result3 = "5" + "3"          # "53" <- string + string => concatenation
```

## Strongly-typed variables

You can constrain a variable to a specific type — assigning an incompatible
value throws an error instead of silently coercing:

```powershell
[int]$score = 95
$score = "not a number"      # throws: Cannot convert value "not a number" to type "System.Int32"
```

## Common types cheat sheet

| Type literal | .NET type | Example |
|--------------|-----------|---------|
| `[int]` | `System.Int32` | `[int]$x = 5` |
| `[double]` | `System.Double` | `[double]$pi = 3.14` |
| `[string]` | `System.String` | `[string]$s = "hi"` |
| `[bool]` | `System.Boolean` | `[bool]$flag = $true` |
| `[array]` | `System.Object[]` | `[array]$a = 1,2,3` |
| `[hashtable]` | `System.Collections.Hashtable` | `[hashtable]$h = @{}` |
| `[datetime]` | `System.DateTime` | `[datetime]$d = Get-Date` |

## Null and $null checks

```powershell
$value = $null

if ($null -eq $value) {
    Write-Output "value is null"
}
```

Always put `$null` on the **left** side of `-eq` when checking — `$value -eq
$null` can misbehave if `$value` happens to be a collection, since PowerShell
then compares element-by-element instead of the whole variable.

## Automatic (built-in) variables

PowerShell pre-populates several variables you'll use constantly:

```powershell
$PSVersionTable.PSVersion   # the running PowerShell version
$PWD                        # current working directory
$HOME                       # user's home directory
$Error[0]                   # the most recent error (see module 08)
$_                          # the current pipeline object inside loops/filters
```

## Variable scope basics

```powershell
$greeting = "outer"

function Show-Greeting {
    Write-Output $greeting     # can READ the outer variable
    $greeting = "inner"        # but this creates a NEW local variable
}

Show-Greeting          # outer
Write-Output $greeting  # outer  <- unchanged outside the function
```

Functions get their own scope by default; assigning to a variable inside a
function does not modify the caller's variable of the same name unless you
explicitly use `$script:` or `$global:`.

```powershell
$counter = 0

function Increment-Counter {
    $script:counter++    # explicitly modify the script-scoped variable
}

Increment-Counter
Write-Output $counter   # 1
```

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `$x = 5` | assign a variable |
| `"$x"` | interpolate inside double quotes |
| `'$x'` | literal — no interpolation |
| `"$($x.Method())"` | sub-expression inside a string |
| `$x.GetType().Name` | get the runtime type name |
| `[int]$x` | cast/constrain to a type |
| `$null -eq $x` | safe null check |
| `$script:x` | reference script scope explicitly |

## How It Actually Works

A PowerShell variable is not a text label pointing at a string — it's an
entry in a `SessionStateScope`'s variable table, stored as a `PSVariable`
object that wraps a live reference to a boxed .NET object on the CLR heap.
When you write `$x = 5`, the engine doesn't store "5"; it creates a boxed
`System.Int32`, wraps it in a `PSObject` (PowerShell's universal object
envelope, which is how every value — even primitives — carries an
`ExtendedTypeSystem` layer of adapted properties/methods), and stores a
reference to that `PSObject` in `$x`'s `PSVariable.Value` slot.

Type coercion at assignment time (`[int]$x = "5"`) goes through the
**LanguagePrimitives.ConvertTo** pipeline, a chain of converters tried in
order: an explicit `IConvertible`/parse method on the target type, a
registered `PSTypeConverter`, then a last-resort reflection-based property
match. This is why `[datetime]"2026-01-01"` works (a registered converter
calls `DateTime.Parse` with culture-aware parsing) while converting to an
unrelated custom type throws `PSInvalidCastException` — there's no
converter in the chain that knows how to bridge the two types.

`$null` is its own singleton instance of `AutomationNull`/`NullVariable`
under the hood, not literally CLR `null` in most contexts — comparisons
like `$null -eq $x` are engine-special-cased so array-unwrapping semantics
(`-eq` against a collection returns matching elements, not a boolean)
don't accidentally swallow null checks, which is also why PowerShell style
guides insist on `$null -eq $x` rather than `$x -eq $null`.

Variable **scope** is implemented as a linked list of `SessionStateScope`
objects, one per function call frame, script block, or module boundary.
Reading a variable walks up the parent-scope chain until it finds a match
(dynamic-like lookup), but *writing* a bare `$x = ...` always creates or
updates it in the **current** scope only — `$script:`, `$global:`, and
`$using:` are explicit scope-modifier prefixes that redirect the write
target to a specific scope object instead of the default local one.

## 🔀 See this in another language

- [Python — Variables, Data Types & Operators](https://sigilipelli.github.io/python-mastery-path/level-1/02-variables-data-types/)
- [C# — Variables & Types](https://sigilipelli.github.io/csharp-mastery-path/level-1/02-variables-types/)
- [Kotlin — Variables & Types](https://sigilipelli.github.io/kotlin-mastery-path/level-1/02-variables-types/)

## Exercise

Write `convert.ps1` that declares a string variable holding `"123"`, casts it
to `[int]`, adds `27` to it, and prints a sentence using string interpolation
that shows both the original string and the final computed number.
