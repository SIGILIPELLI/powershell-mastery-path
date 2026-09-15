---
description: "Control Flow — PowerShell uses named comparison operators instead of symbols like ) and pipeline syntax."
---

# 03 · Control Flow

## if / elseif / else

```powershell
$age = 20

if ($age -lt 13) {
    Write-Output "child"
} elseif ($age -lt 20) {
    Write-Output "teenager"
} else {
    Write-Output "adult"
}
```

```text
adult
```

PowerShell uses named comparison operators instead of symbols like `<` or
`==` — this avoids clashing with redirection (`<`, `>`) and pipeline syntax.

## Comparison operators cheat sheet

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

## Logical operators

```powershell
$age = 25
$hasLicense = $true

if ($age -ge 18 -and $hasLicense) {
    Write-Output "can drive"
}

if ($age -lt 18 -or -not $hasLicense) {
    Write-Output "cannot drive"
} else {
    Write-Output "can drive"
}
```

## switch statement

`switch` in PowerShell is much more powerful than a typical C-style switch —
it supports wildcards, regex, and script-block conditions, and it can operate
directly on a collection.

```powershell
$fruit = "banana"

switch ($fruit) {
    "apple"  { Write-Output "It's an apple" }
    "banana" { Write-Output "It's a banana" }
    default  { Write-Output "Unknown fruit" }
}
```

```powershell
# switch falls through to every matching case unless you use 'break'
$number = 4

switch ($number) {
    { $_ -lt 5 }  { Write-Output "less than 5" }
    { $_ -eq 4 }  { Write-Output "exactly 4" }
    { $_ -gt 2 }  { Write-Output "greater than 2" }
}
# less than 5
# exactly 4
# greater than 2      <- all three matched, all three ran
```

```powershell
# switch can iterate an entire array directly
$scores = 55, 72, 91, 40

switch ($scores) {
    { $_ -ge 90 } { Write-Output "$_ : A" }
    { $_ -ge 70 } { Write-Output "$_ : B" }
    default       { Write-Output "$_ : F" }
}
```

```powershell
# -Wildcard and -Regex modes
switch -Wildcard ($fruit) {
    "b*" { Write-Output "starts with b" }
}

switch -Regex ("user42") {
    '^\D+\d+$' { Write-Output "letters followed by digits" }
}
```

## Loops

```powershell
# for
for ($i = 1; $i -le 3; $i++) {
    Write-Output "i = $i"
}
```

```powershell
# while
$n = 3
while ($n -gt 0) {
    Write-Output "countdown: $n"
    $n--
}
```

```powershell
# do/while — always runs the body at least once
$attempts = 0
do {
    $attempts++
    Write-Output "attempt $attempts"
} while ($attempts -lt 3)
```

```powershell
# foreach — iterate any collection
$colors = "red", "green", "blue"
foreach ($color in $colors) {
    Write-Output "color: $color"
}
```

```powershell
# ForEach-Object — the pipeline version, used with piped input (see module 05)
1..3 | ForEach-Object { Write-Output "piped value: $_" }
```

## break and continue

```powershell
foreach ($n in 1..10) {
    if ($n -eq 3) { continue }   # skip this iteration
    if ($n -eq 6) { break }      # stop the loop entirely
    Write-Output $n
}
# 1
# 2
# 4
# 5
```

## Cheat sheet

| Construct | Use case |
|-----------|----------|
| `if / elseif / else` | branch on a condition |
| `switch` | branch on multiple values, wildcards, or regex |
| `for` | counted loop |
| `while` | condition-checked-first loop |
| `do { } while` | condition-checked-after loop (runs once minimum) |
| `foreach ($x in $collection)` | iterate an in-memory collection |
| `ForEach-Object` | iterate piped pipeline input |
| `break` / `continue` | exit loop / skip to next iteration |

## How It Actually Works

Every `if`, `switch`, and loop keyword in PowerShell compiles down through
the same AST node types — `IfStatementAst`, `SwitchStatementAst`,
`PipelineAst` inside `WhileStatementAst`, and so on — that the parser
builds from your script text before a single instruction runs. This
two-phase design (parse fully, then execute) is why a syntax error deep
inside an `if` branch that never runs is still caught immediately: the
whole AST has to be valid before execution starts, unlike a naive
line-by-line interpreter.

`switch` is more than sugar for `if/elseif` chains — its default mode
evaluates **every** case top to bottom against the subject (falling
through unless you `break`), and each case label is itself a full
expression, not just a literal: `switch ($x) { {$_ -gt 10} {...} }`
compiles the `{...}` script block as a predicate and calls it with `$_`
bound to the subject via `$PSItem`/`$_` — this is how `-Regex`, `-Wildcard`,
and script-block cases can all coexist in one construct. Internally the
switch statement evaluator tries, per case in order: exact/type match,
wildcard match (if `-Wildcard`), regex match (if `-Regex`, compiling a
`System.Text.RegularExpressions.Regex` per case), or script-block
invocation, short-circuiting to the next case only when the current one
returns `$false`.

Loops (`foreach`, `while`, `for`, `do/while`) each lower to a
`LoopStatementAst` with a condition pipeline and body script block; the
runtime re-enters the body's own child scope on every iteration for
`foreach`, which is why a `function` or `$using:` closure captured inside
a loop body captures the *current* iteration's binding, not a shared one —
PowerShell doesn't have the "closures share the loop variable" trap that
some C-family languages have with `var`. `break` and `continue` are
implemented as special control-flow exceptions (`BreakException`,
`ContinueException`) that unwind the call stack until caught by the
nearest enclosing loop or labeled block — which is exactly why labeled
`break Outer` works across nested loops: it's a targeted exception with a
label match, not a jump instruction.

## 🔀 See this in another language

- [Python — Control Flow](https://sigilipelli.github.io/python-mastery-path/level-1/03-control-flow/)
- [C# — Control Flow](https://sigilipelli.github.io/csharp-mastery-path/level-1/03-control-flow/)
- [Kotlin — Control Flow](https://sigilipelli.github.io/kotlin-mastery-path/level-1/03-control-flow/)

## Exercise

Write `fizzbuzz.ps1` that loops from 1 to 30 using a `for` loop and, for each
number, uses a `switch` statement (not chained `if`) to print `"Fizz"` for
multiples of 3, `"Buzz"` for multiples of 5, `"FizzBuzz"` for multiples of
both, and the number itself otherwise.
