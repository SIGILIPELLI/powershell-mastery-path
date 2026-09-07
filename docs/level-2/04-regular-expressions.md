# 04 · Regular Expressions in PowerShell

Wildcards (`-like`) can only match simple patterns like `*.txt`. Regular
expressions let you match structure — "three digits, a dash, then four
digits," or "anything that looks like an email address" — and PowerShell
has first-class operators for them built on .NET's regex engine, which is
considerably more powerful than the POSIX regex most shells use.

## `-match`: the basic regex test

```powershell
"user-4021" -match "^user-\d+$"      # True
"admin-x"   -match "^user-\d+$"      # False
```

`-match` returns `$true`/`$false`, and as a side effect populates the
automatic `$Matches` hashtable with the match details — even though you
didn't ask for them explicitly.

```powershell
if ("2026-08-02" -match "(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})") {
    Write-Output "Year: $($Matches['year'])"
    Write-Output "Month: $($Matches['month'])"
}
```

```text
Year: 2026
Month: 08
```

`(?<year>...)` is a **named capture group** — instead of remembering that
group 1 is the year, you reference it by name through `$Matches['year']`.
`$Matches[0]` always holds the entire matched substring.

## The trap: `$Matches` is stale until the next successful match

```powershell
"abc" -match "\d+"        # False — no digits in "abc"
Write-Output $Matches[0]   # still holds the PREVIOUS match, not cleared!
```

`-match` only updates `$Matches` **on a successful match** — a failed
`-match` leaves the old value sitting there, which silently produces wrong
results if you don't check the boolean return value first. Always guard
access to `$Matches` behind the `if (... -match ...)` check itself, never
read it unconditionally afterward.

Note also that the whole-match key is the **integer** `0`, not the string
`"0"` — `$Matches[0]` works, but `$Matches['0']` returns nothing. Named
groups don't have this trap: `$Matches['year']` works fine because named
keys really are strings.

## Case sensitivity: `-match` vs `-cmatch`

```powershell
"HELLO" -match "hello"     # True  -- -match is case-INsensitive by default
"HELLO" -cmatch "hello"    # False -- -cmatch forces case sensitivity
```

Every comparison operator in PowerShell has this pattern: the plain form
(`-match`, `-eq`, `-replace`, `-like`) is case-insensitive, and prefixing
`c` (`-cmatch`, `-ceq`, `-creplace`, `-clike`) makes it case-sensitive.
Prefixing `i` (`-imatch`) is also valid and explicit but redundant with the
default.

## `-notmatch`: the inverse test

```powershell
$logLines = "INFO: startup ok", "ERROR: disk full", "INFO: request served"
$logLines | Where-Object { $_ -notmatch "^ERROR" }
```

```text
INFO: startup ok
INFO: request served
```

## `-replace`: regex-powered substitution

```powershell
"2026-08-02" -replace "-", "/"
# 2026/08/02

"Call 555-1234 or 555-5678" -replace "\d{3}-\d{4}", "[REDACTED]"
# Call [REDACTED] or [REDACTED]
```

`-replace` always treats its pattern as a regex — if you need a **literal**
string replacement (e.g. replacing a literal `.` without it meaning "any
character"), escape the pattern with `[regex]::Escape(...)`:

```powershell
$literalDot = [regex]::Escape(".")
"3.14.15" -replace $literalDot, "_"
# 3_14_15
```

### Back-references in replacement text

```powershell
"John Smith" -replace "(\w+)\s(\w+)", '$2, $1'
# Smith, John
```

`$1`, `$2`, etc. in the replacement string refer to captured groups from the
pattern — use single quotes for the replacement string so PowerShell
doesn't try to interpolate `$1` as a variable itself.

## `Select-String`: regex search across files/lines

```powershell
Select-String -Path "app.log" -Pattern "ERROR|WARN"
```

```text
app.log:12:ERROR: connection refused
app.log:45:WARN: retrying request
```

```powershell
$matches = Select-String -Path "app.log" -Pattern "user (?<id>\d+) logged in"
foreach ($m in $matches) {
    Write-Output "Line $($m.LineNumber): user $($m.Matches.Groups['id'].Value)"
}
```

`Select-String` is PowerShell's equivalent of `grep` — it returns
`MatchInfo` objects with `.LineNumber`, `.Line`, and `.Matches` (a full
.NET `Match` collection), so you get structured results instead of raw
text lines.

## Going deeper: the `[regex]` .NET class directly

For anything beyond a quick test/replace — like getting **all** matches in
a string, not just the first — drop down to the `[regex]` type directly.

```powershell
$text = "order #1042 shipped, order #1077 pending, order #1090 shipped"

[regex]::Matches($text, "#(\d+)") | ForEach-Object {
    $_.Groups[1].Value
}
```

```text
1042
1077
1090
```

`-match` only ever finds the **first** match in a string; `[regex]::Matches`
returns all of them. This is the single most common regex mistake in
PowerShell scripts — using `-match` in a loop expecting multiple hits from
one string, when it will only ever report one.

## Common patterns

| Pattern | Matches |
|---|---|
| `\d+` | one or more digits |
| `\w+` | one or more word characters (letters, digits, `_`) |
| `\s+` | one or more whitespace characters |
| `^...$` | anchors to start/end of the string |
| `.*?` | non-greedy "anything" (stops at the first possible match) |
| `(?<name>...)` | named capture group |
| `a{2,4}` | between 2 and 4 repetitions of `a` |
| `[A-Z]{2}\d{4}` | e.g. an ID like `EN2045` |

## Cheat sheet

| Operator/Tool | Purpose |
|---|---|
| `-match` / `-notmatch` | test a string against a regex, populates `$Matches` |
| `-cmatch` | case-sensitive version of `-match` |
| `-replace` | regex-powered find-and-replace |
| `-creplace` | case-sensitive version of `-replace` |
| `$Matches['name']` | read a named capture group after a successful `-match` |
| `Select-String` | grep-like search across strings/files, returns rich objects |
| `[regex]::Matches($s, $p)` | get **every** match, not just the first |
| `[regex]::Escape($s)` | treat a string as a literal pattern, not regex syntax |

## How It Actually Works

PowerShell's regex operators (`-match`, `-replace`, `-split` in regex
mode) are thin wrappers over .NET's `System.Text.RegularExpressions.Regex`
engine, which is a **backtracking NFA (nondeterministic finite automaton)
simulator**, not a linear-time DFA matcher like `grep -E`'s engine.
Concretely: the engine walks the pattern's compiled instruction tree
against the input, and whenever a branch fails to continue matching, it
backtracks to the last choice point and tries the next alternative. This
is why "catastrophic backtracking" is a real, reproducible performance
bug in PowerShell regexes — a pattern like `(a+)+b` against a long string
of `a`s with no trailing `b` causes exponential blowup in the number of
backtracking states explored, and it's the exact same failure mode Perl,
Java, and JavaScript regex engines share, because they're all
backtracking engines too.

`-match` sets the automatic `$Matches` variable as a side effect of a
successful match — it's populated from the `Regex.Match()` call's
`GroupCollection`, keyed by group number for unnamed captures and by name
for `(?<name>...)` named groups, which is why `$Matches.name` works
directly as a hashtable lookup rather than needing index arithmetic.
`-replace` with a script-block-like substitution isn't supported directly
in the operator form, but the underlying `[regex]::Replace()` with a
`MatchEvaluator` delegate (available when you drop to the .NET type
directly) is how you can substitute based on per-match computed logic
that the simple `-replace 'pattern', 'replacement'` string-template form
can't express.

`-split` in regex mode compiles the delimiter pattern once and calls
`Regex.Split()`, which — unlike simple string `.Split()` — evaluates the
pattern against the whole input in one pass and additionally captures any
parenthesized delimiter groups *into* the result array positions between
elements, a detail that surprises people expecting only the non-delimiter
segments back.

## Exercise

Write a script that takes an array of log lines like
`"2026-08-02 10:15:03 ERROR Failed to connect to db"` and, using a single
regex with named capture groups (`date`, `time`, `level`, `message`), builds
a `[pscustomobject]` for each line with those four properties. Filter the
resulting objects to only `level -eq "ERROR"` and print them as a table.
