---
description: "Working with Files — Prefer Set-Content/Add-Content for plain text and Export-Csv/ ConvertTo-Json | Set-Content for structured data; use Out-File when you…"
---

# 07 · Working with Files

## Checking paths

```powershell
Test-Path "notes.txt"          # $true or $false
Test-Path "C:\missing\dir"      # $false
```

## Reading files

```powershell
# Get-Content reads a file as an array of lines
$lines = Get-Content -Path "notes.txt"
$lines.Count
$lines[0]           # first line

foreach ($line in $lines) {
    Write-Output "Line: $line"
}
```

```powershell
# -Raw reads the whole file as a single string (preserves original newlines)
$wholeFile = Get-Content -Path "notes.txt" -Raw
$wholeFile.Length
```

```powershell
# reading large files efficiently, one line at a time
Get-Content -Path "big.log" | ForEach-Object {
    if ($_ -match "ERROR") {
        Write-Output $_
    }
}
```

## Writing files

```powershell
"First line" | Set-Content -Path "output.txt"     # overwrites the file
"Second line" | Add-Content -Path "output.txt"     # appends to the file

Get-Content "output.txt"
# First line
# Second line
```

```powershell
# writing multiple lines at once
$lines = "line 1", "line 2", "line 3"
$lines | Set-Content -Path "output.txt"

# writing structured data as JSON
@{ name = "Ada"; age = 36 } | ConvertTo-Json | Set-Content -Path "person.json"
```

## Out-File vs Set-Content

```powershell
"hello" | Out-File -FilePath "a.txt"       # designed for general "output to file" formatting
"hello" | Set-Content -Path "b.txt"        # designed for text content, more predictable encoding
```

Prefer `Set-Content`/`Add-Content` for plain text and `Export-Csv`/
`ConvertTo-Json | Set-Content` for structured data; use `Out-File` when you
specifically want the same formatted output that would print to console.

## Working with paths

```powershell
$path = "/Users/ada/reports/2026/summary.txt"

Split-Path -Path $path -Leaf         # summary.txt
Split-Path -Path $path -Parent        # /Users/ada/reports/2026
[System.IO.Path]::GetExtension($path) # .txt
[System.IO.Path]::GetFileNameWithoutExtension($path)  # summary

Join-Path -Path "/Users/ada" -ChildPath "reports"      # /Users/ada/reports
```

## Directories

```powershell
New-Item -Path "logs" -ItemType Directory -Force   # create (no error if it already exists)

Get-ChildItem -Path "." -Directory                  # list subdirectories
Get-ChildItem -Path "." -File                         # list files only
Get-ChildItem -Path "." -Recurse -Filter "*.ps1"      # find files recursively
```

```powershell
# copying, moving, removing
Copy-Item -Path "notes.txt" -Destination "backup/notes.txt"
Move-Item -Path "old.txt" -Destination "archive/old.txt"
Remove-Item -Path "temp.txt"
Remove-Item -Path "temp-folder" -Recurse -Force
```

## File metadata

```powershell
$file = Get-Item -Path "notes.txt"

$file.Length         # size in bytes
$file.LastWriteTime   # last modified timestamp
$file.CreationTime     # creation timestamp
$file.Extension        # .txt
```

## Putting it together: a simple log scanner

```powershell
$logPath = "app.log"

if (-not (Test-Path $logPath)) {
    Write-Output "No log file found at $logPath"
    return
}

$errorLines = Get-Content -Path $logPath | Where-Object { $_ -match "ERROR" }

Write-Output "Found $($errorLines.Count) error line(s):"
$errorLines | ForEach-Object { Write-Output " - $_" }
```

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `Test-Path` | check whether a file/directory exists |
| `Get-Content` | read a file (array of lines, or `-Raw` for one string) |
| `Set-Content` / `Add-Content` | overwrite / append text |
| `New-Item -ItemType Directory` | create a directory |
| `Get-ChildItem` | list files/directories (`ls`/`dir` alias) |
| `Copy-Item` / `Move-Item` / `Remove-Item` | copy / move / delete |
| `Get-Item` | get metadata about a file/directory |
| `Split-Path` / `Join-Path` | decompose / build file paths |

## How It Actually Works

File cmdlets in PowerShell don't talk to the filesystem directly — they go
through the **PSProvider/PSDrive abstraction**. `Get-ChildItem`,
`Get-Content`, `Set-Content`, `Test-Path`, and friends are all
*provider-agnostic* cmdlets; the actual work is delegated to whichever
`CmdletProvider` owns the drive the path resolves to. `C:\`, `/home/user`,
and a mapped network share all route through the built-in `FileSystem`
provider, but the exact same cmdlets also work unmodified against the
`Registry:`, `Cert:`, `Env:`, `Function:`, and `Variable:` providers —
`Get-ChildItem Env:` and `Get-ChildItem C:\Temp` are literally the same
cmdlet dispatching to two different provider implementations of
`GetChildItems`. This is why PowerShell calls filesystem locations
"drives" even when there's no physical drive involved.

`Get-Content` reads the whole file's encoding-aware text by default and
returns it as an **array of strings, one per line** (or one giant string
with `-Raw`), because internally it wraps a `StreamReader` and calls
`ReadLine()` in a loop, emitting each line as a separate pipeline object —
this is exactly why `(Get-Content file.txt).Count` gives you a line count
for free and why piping `Get-Content` into `Where-Object` filters
line-by-line rather than needing manual splitting.

`Out-File`/`Set-Content` differ in more than convenience: `Set-Content`
writes strings as strings via the provider's content-writer interface
with `Encoding` control and is optimized as a straight write; `Out-File`
first pushes every object through the formatting subsystem (the same
`Format-*` engine behind console output) producing formatted display text
*before* writing — which is why redirecting complex objects with `>` /
`Out-File` can silently truncate columns to console width while
`ConvertTo-Json | Set-Content` preserves full structure: one path goes
through the formatter, the other doesn't.

## 🔀 See this in another language

- [Python — File I/O Basics](https://sigilipelli.github.io/python-mastery-path/level-1/07-file-io/)
- [C# — Exception Handling](https://sigilipelli.github.io/csharp-mastery-path/level-1/07-exception-handling/)
- [Kotlin — Collections](https://sigilipelli.github.io/kotlin-mastery-path/level-1/07-collections/)

## Exercise

Write `word-count.ps1` that reads a text file path from a parameter, checks
it exists with `Test-Path` (printing an error and exiting if not), then
reports the number of lines (`Get-Content` array count) and the total number
of words across the whole file (split each line on whitespace and sum the
counts).
