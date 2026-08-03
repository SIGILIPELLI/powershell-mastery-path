# 09 · Working with CSV/Excel Data

CSV files are everywhere in real automation work — exported reports,
data dumps from other systems, input for bulk operations. This module
covers `Import-Csv`/`Export-Csv` properly, the single biggest trap they
share (everything comes back as a string), and how to go a step further
into real `.xlsx` files with the community `ImportExcel` module.

## Export-Csv: objects to a file

```powershell
$employees = @(
    [pscustomobject]@{ Name = "Priya"; Department = "Engineering"; Salary = 95000 }
    [pscustomobject]@{ Name = "Diego"; Department = "Sales"; Salary = 72000 }
    [pscustomobject]@{ Name = "Amara"; Department = "Engineering"; Salary = 101000 }
)

$employees | Export-Csv -Path ./employees.csv -NoTypeInformation
Get-Content ./employees.csv
```

```text
"Name","Department","Salary"
"Priya","Engineering","95000"
"Diego","Sales","72000"
"Amara","Engineering","101000"
```

`-NoTypeInformation` suppresses an extra `#TYPE ...` header line that older
PowerShell versions used to add — always include it; modern PowerShell (7+)
doesn't add that header by default, but specifying it explicitly keeps
scripts portable across versions and makes the intent clear.

## Import-Csv: file back to objects

```powershell
$imported = Import-Csv -Path ./employees.csv
$imported | Format-Table -AutoSize

$imported[0].GetType().Name        # PSCustomObject
$imported[0].Salary.GetType().Name  # String  <-- not Int32!
```

```text
Name  Department  Salary
----  ----------  ------
Priya Engineering 95000
Diego Sales       72000
Amara Engineering 101000

PSCustomObject
String
```

## The trap: every CSV value comes back as a string

CSV is a text format — `Import-Csv` has no way to know `"95000"` was
originally a number, so **every single property comes back as a string**,
even ones that look numeric. This causes two very different kinds of bugs.

### Comparison trap

```powershell
"9000" -gt "72000"                  # True  -- WRONG: comparing text lexicographically
9000 -gt 72000                       # False -- correct, real numbers
[int]"9000" -gt [int]"72000"        # False -- correct, cast first
```

String comparison looks at characters left to right: `"9"` sorts after
`"7"`, so `"9000"` is considered "greater than" `"72000"` even though 9000
is numerically smaller — a classic string-vs-number surprise that silently
produces wrong results instead of an error.

### Sorting trap

```powershell
# WRONG: sorts as text, not by magnitude
$imported | Sort-Object Salary -Descending

# CORRECT: cast inside a calculated expression first
$imported | Sort-Object { [int]$_.Salary } -Descending
```

### The fix: cast explicitly, every time

```powershell
$total = ($imported | ForEach-Object { [int]$_.Salary } | Measure-Object -Sum).Sum
Write-Output "Total payroll: $total"
```

```text
Total payroll: 268000
```

Handy exception: `Measure-Object -Sum`/`-Average` is smart enough to
coerce numeric-looking strings automatically, so
`$imported | Measure-Object -Property Salary -Sum` also works without an
explicit cast — but don't rely on that elsewhere; `Sort-Object` and
comparison operators do **not** do this coercion for you.

## Filtering and grouping imported data

```powershell
$imported |
    Where-Object Department -eq "Engineering" |
    Sort-Object { [int]$_.Salary } -Descending |
    Format-Table -AutoSize
```

```text
Name  Department  Salary
----  ----------  ------
Amara Engineering 101000
Priya Engineering 95000
```

Everything from [Module 01](01-advanced-pipeline.md)'s pipeline techniques
applies identically to imported CSV data — the only adjustment is
remembering to cast numeric-looking columns before comparing or sorting
them.

## Custom delimiters

```powershell
$employees | Export-Csv -Path ./employees_semicolon.csv -NoTypeInformation -Delimiter ";"
Import-Csv -Path ./employees_semicolon.csv -Delimiter ";"
```

Some locales and tools (particularly European Excel configurations) use
`;` instead of `,` as the default CSV separator — `-Delimiter` on both
cmdlets keeps you compatible either way.

## Appending to an existing CSV

```powershell
[pscustomobject]@{ Name = "Kenji"; Department = "Support"; Salary = 68000 } |
    Export-Csv -Path ./employees.csv -NoTypeInformation -Append
```

`-Append` adds a row without rewriting the header — useful for logging
scripts that add one row per run rather than regenerating the whole file
each time. Without `-Append`, `Export-Csv` overwrites the file completely.

## Going further: real Excel files with `ImportExcel`

`Export-Csv`/`Import-Csv` only handle plain-text CSV — for actual `.xlsx`
workbooks (multiple sheets, formatting, formulas), the community
`ImportExcel` module is the standard tool; it doesn't require Excel itself
to be installed.

```powershell
Install-Module -Name ImportExcel -Scope CurrentUser -Force
Import-Module ImportExcel

$employees | Export-Excel -Path ./employees.xlsx -WorksheetName "Staff" -TableName "StaffTable"

$fromExcel = Import-Excel -Path ./employees.xlsx -WorksheetName "Staff"
$fromExcel | Format-Table -AutoSize
$fromExcel[0].Salary.GetType().Name    # Double  <-- Excel keeps the real numeric type
```

```text
Name  Department  Salary
----  ----------  ------
Priya Engineering 95000
Diego Sales       72000
Amara Engineering 101000

Double
```

This is the payoff for the extra module: unlike CSV, `.xlsx` cells carry a
real data type, so numbers come back as `Double`/`Int32`, not strings — no
casting trap to remember. `-TableName` also formats the data as a proper
Excel table (filterable headers, banded rows) rather than plain cell
values.

## Cheat sheet

| Cmdlet/Parameter | Purpose |
|---|---|
| `Export-Csv -NoTypeInformation` | write objects to CSV (always include this switch) |
| `Import-Csv` | read CSV rows back as objects — **all properties are strings** |
| `-Delimiter ";"` | use a non-comma separator |
| `-Append` | add rows without rewriting the header |
| `[int]$_.Property` | cast before sorting/comparing numeric-looking CSV columns |
| `Measure-Object -Sum` | one of the few cmdlets that auto-coerces numeric strings |
| `Export-Excel` / `Import-Excel` (ImportExcel module) | real `.xlsx` files with proper types |

## Exercise

Create a CSV `inventory.csv` with columns `Item`, `Quantity`, `UnitPrice`
(a handful of rows, values as plain numbers). Write a script that imports
it, casts `Quantity` and `UnitPrice` to numeric types, adds a calculated
`TotalValue` property (`Quantity * UnitPrice`) to each row using
`Select-Object` with a calculated property, sorts descending by
`TotalValue`, and exports the result to `inventory-report.csv` with
`Export-Csv -NoTypeInformation`.
