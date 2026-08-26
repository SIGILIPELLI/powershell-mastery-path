# 03 · Cross-Platform PowerShell

PowerShell 7 runs on Windows, Linux, and macOS from the same executable —
but "runs everywhere" isn't the same as "behaves identically everywhere."
Paths, line endings, case sensitivity, and even which aliases exist all
differ by platform, and scripts that don't account for that break the
moment they leave the machine they were written on.

## Detecting the platform

```powershell
$PSVersionTable.PSVersion
$IsWindows, $IsLinux, $IsMacOS
```

```text
Major Minor Patch PreReleaseLabel BuildLabel
----- ----- ----- --------------- ----------
7     6     4

False
False
True
```

`$IsWindows`, `$IsLinux`, `$IsMacOS` are automatic boolean variables
available in PowerShell 7+ (they don't exist in Windows PowerShell
5.1, where everything is implicitly Windows) — the standard way to branch
platform-specific logic without shelling out to `uname` or parsing
`$PSVersionTable.OS`.

```powershell
if ($IsWindows) {
    $configDir = "$env:APPDATA\MyApp"
} else {
    $configDir = "$HOME/.config/myapp"
}
```

## The trap: hardcoded backslash paths

```powershell
$badPath = "C:\Users\me\data.txt"          # breaks on Linux/macOS entirely
$goodPath = Join-Path (Join-Path $HOME "data") "data.txt"
"Good path: $goodPath"
```

```text
Good path: /Users/bhanuja/data/data.txt
```

```powershell
[System.IO.Path]::DirectorySeparatorChar
Join-Path "folder" "file.txt"
```

```text
/
folder/file.txt
```

`Join-Path` (and `[System.IO.Path]` methods generally) always emit the
correct separator for the platform actually running the script.
`$HOME` is populated on all three platforms — prefer it over
`$env:USERPROFILE` (Windows-only) or `$env:HOME` (Unix-only) directly.
Any literal `\` or `/` typed into a path string is a portability bug
waiting to surface the first time the script runs somewhere else.

## The trap: line endings

```powershell
$text = "line1`r`nline2`nline3"
($text -split '\r?\n').Count
```

```text
3
```

Windows-authored text files traditionally use `\r\n`; Linux/macOS use
`\n`. A file downloaded or generated on one platform and processed with a
line-ending assumption baked in (`-split "`n"` alone, missing files with
`\r\n`) silently mis-splits or leaves stray `\r` characters trailing each
line. `-split '\r?\n'` handles both correctly regardless of where the text
came from — worth using by default for any text you didn't generate
yourself in the same script.

## The trap: filesystem case sensitivity varies by OS *and* by filesystem

```powershell
New-Item -ItemType Directory -Path "/tmp/psl/CaseTest" -Force | Out-Null
"Hello" | Out-File "/tmp/psl/CaseTest/File.txt"
Test-Path "/tmp/psl/CaseTest/file.txt"
```

```text
True
```

That returned `True` on this machine (macOS, APFS, case-insensitive by
default) — the same script on a typical Linux filesystem (ext4,
case-sensitive) would return `False` for the same input, because
`file.txt` and `File.txt` are different files there. Windows (NTFS) is
case-insensitive like macOS by default. The trap: code that "works" on
your Mac or Windows box because the filesystem quietly tolerates a case
mismatch can fail the moment it runs on Linux CI or a Linux server —
always write paths with the exact case you mean, and never rely on the
filesystem to paper over a typo.

## Native command exit codes: `$LASTEXITCODE`

```powershell
& ls /nonexistent-path-xyz 2>$null
"Exit code: $LASTEXITCODE"
```

```text
Exit code: 1
```

`$LASTEXITCODE` reflects the exit code of the last **native** (external)
command run — not a PowerShell cmdlet's success/failure, which is
`$?`/exceptions instead. Any script that shells out to a native tool
(`git`, `docker`, `curl`, platform package managers) and needs to detect
failure has to check `$LASTEXITCODE` explicitly; PowerShell doesn't throw
automatically just because the external process returned nonzero.

## The trap: aliases that only exist on some platforms

```powershell
Get-Alias ls -ErrorAction SilentlyContinue
Get-Command ls -All | Select-Object CommandType, Source
```

```text
CommandType Source
----------- ------
Application /bin/ls
```

On this machine, `ls` resolves straight to the real Unix `/bin/ls`
binary — there's no PowerShell alias named `ls` shadowing it here. On
Windows PowerShell, `ls` **is** an alias for `Get-ChildItem`, which
accepts different parameters entirely (`-Force` means something different
to each). A script that calls `ls -la` assuming the Unix binary, or calls
`ls -Recurse` assuming the `Get-ChildItem` alias, is making a platform
assumption that silently breaks on the other kind of machine — this is
exactly why scripts meant to be portable should call `Get-ChildItem`
directly rather than `ls`, `dir`, or any alias whose target varies.

## Cheat sheet

| Concern | Cross-platform fix |
|---|---|
| Which OS is this? | `$IsWindows` / `$IsLinux` / `$IsMacOS` |
| Building a path | `Join-Path`, never a literal `\` or `/` |
| Home directory | `$HOME` (works everywhere; avoid `$env:USERPROFILE`/`$env:HOME` directly) |
| Splitting text into lines | `-split '\r?\n'`, not just `"`n"` or `"`r`n"` |
| Filesystem case sensitivity | assume case-sensitive always, regardless of what your dev machine tolerates |
| Native command success/failure | check `$LASTEXITCODE`, not `$?` |
| `ls`, `dir`, other aliases | call `Get-ChildItem` directly in portable scripts |
| Environment variables | `$env:PATH` uses `[System.IO.Path]::PathSeparator` to split (`:` vs `;`) |

## Exercise

Take a script that assumes Windows (uses `$env:USERPROFILE`, backslash
paths, and `dir` for listing files) and rewrite it to run identically on
Windows, Linux, and macOS: swap in `$HOME`, `Join-Path`, `Get-ChildItem`,
and an explicit `$IsWindows`/`$IsLinux`/`$IsMacOS` branch anywhere
platform-specific behavior is unavoidable (like a config directory path).
Run it with `pwsh -File` and confirm it produces sensible output on
whatever platform you have available.
