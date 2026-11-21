---
title: "Final Quiz"
description: "Test what you have learned across all four levels of PowerShell Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```powershell\n$a = 1,2,3\n$a -eq 2\n$a -contains 2\n$null -eq $a\n(1..3 | ForEach-Object { $_ * $_ }) -join \",\"\n```\n",
    "options": [
      "2\nTrue\nFalse\n1,4,9",
      "True\nTrue\nFalse\n1,4,9",
      "Error",
      "2\n2\nFalse\n1,4,9"
    ],
    "correctIndex": 0,
    "explanation": "-eq on an array filters; -contains tests membership. Verified by running it in PowerShell 7."
  },
  {
    "question": "What does this print?\n```powershell\nfunction f { param([int]$n) $n * 2 }\nf \"21\"\ntry { f \"abc\" } catch { \"bad\" }\n```\n",
    "options": [
      "42\nbad",
      "Error",
      "21\nbad",
      "42\n0"
    ],
    "correctIndex": 0,
    "explanation": "Typed parameters convert or throw. Verified by running it in PowerShell 7."
  },
  {
    "question": "What does this print?\n```powershell\n$h = @{ a = 1; b = 2 }\n$h.GetEnumerator() | Sort-Object Name | ForEach-Object { \"$($_.Name)$($_.Value)\" }\n($h.Keys | Measure-Object).Count\n```\n",
    "options": [
      "a1\nb2\n2",
      "Error",
      "b2\na1\n2",
      "a1 b2\n2"
    ],
    "correctIndex": 0,
    "explanation": "Hashtable enumeration. Verified by running it in PowerShell 7."
  },
  {
    "question": "What does this print?\n```powershell\n$ErrorActionPreference = 'Continue'\nGet-Item /nope_zzz 2>$null\n\"next\"\ntry { Get-Item /nope_zzz -ErrorAction Stop } catch { \"caught\" }\n```\n",
    "options": [
      "Error",
      "next\ncaught",
      "next",
      "caught\nnext"
    ],
    "correctIndex": 1,
    "explanation": "Non-terminating errors continue; -ErrorAction Stop makes them catchable. Verified by running it in PowerShell 7."
  },
  {
    "question": "What does this print?\n```powershell\n\"{0:N1} {1:P0} {2:D3}\" -f 3.14159, 0.256, 7\n'a,b,,c'.Split(',').Count\n('x' * 3) + 'y'\n\"Hello\".Substring(1,3)\n```\n",
    "options": [
      "3.1 0.256 007\n3\nxxxy\nell",
      "3.1 26% 007\n4\nxxxy\nell",
      "3.14 26% 007\n4\nxxxy\nell",
      "Error"
    ],
    "correctIndex": 1,
    "explanation": "Format operator and string methods. Verified by running it in PowerShell 7."
  },
  {
    "question": "What does this print?\n```powershell\n$o = [pscustomobject]@{ A = 1 }\n$o | Add-Member -NotePropertyName B -NotePropertyValue 2\n($o | Get-Member -MemberType NoteProperty).Name -join \",\"\n$o.B * 5\n```\n",
    "options": [
      "A,B\n2",
      "A,B\n10",
      "A\n10",
      "Error"
    ],
    "correctIndex": 1,
    "explanation": "Objects are extensible. Verified by running it in PowerShell 7."
  },
  {
    "question": "Why prefer returning objects over Write-Host text?",
    "options": [
      "Style",
      "Callers can filter, sort and export objects; host text is for display only",
      "Speed",
      "Law"
    ],
    "correctIndex": 1,
    "explanation": "Composable scripts."
  },
  {
    "question": "A script must run unattended nightly. What is essential?",
    "options": [
      "Logging, error handling with non-zero exit codes and least-privilege credentials",
      "Admin",
      "A prompt",
      "Nothing"
    ],
    "correctIndex": 0,
    "explanation": "Unattended reliability."
  },
  {
    "question": "When does Get-Content huge.log | Select-Object -First 5 stop reading?",
    "options": [
      "After sort",
      "As soon as 5 lines are received (the pipeline signals upstream to stop)",
      "Never",
      "After file"
    ],
    "correctIndex": 1,
    "explanation": "Streaming."
  },
  {
    "question": "Which cmdlet adds safety for destructive actions?",
    "options": [
      "-Verbose",
      "-Force",
      "-Quiet",
      "-WhatIf / -Confirm via SupportsShouldProcess"
    ],
    "correctIndex": 3,
    "explanation": "Preview first."
  },
  {
    "question": "Remote results are missing methods. Why?",
    "options": [
      "Firewall",
      "Objects are serialized and come back deserialized (properties only)",
      "Version",
      "Bug"
    ],
    "correctIndex": 1,
    "explanation": "Call methods remotely."
  },
  {
    "question": "A module's helper functions leak to users. Fix?",
    "options": [
      "Export only public functions with Export-ModuleMember or the manifest FunctionsToExport",
      "Rename",
      "Delete",
      "Alias"
    ],
    "correctIndex": 0,
    "explanation": "Small public API."
  },
  {
    "question": "A native tool fails but the script continues. What do you check?",
    "options": [
      "Verbose",
      "$LASTEXITCODE after calling it (and use throw)",
      "$Error",
      "$?"
    ],
    "correctIndex": 1,
    "explanation": "Native commands don't throw."
  },
  {
    "question": "Which Pester feature isolates a test from the network?",
    "options": [
      "Retry",
      "Tag",
      "Mock the command that calls out",
      "Skip"
    ],
    "correctIndex": 2,
    "explanation": "Deterministic tests."
  },
  {
    "question": "How avoid secrets in scripts?",
    "options": [
      "Base64",
      "Plain text",
      "Use SecretManagement/vaults or CI secret variables",
      "Comments"
    ],
    "correctIndex": 2,
    "explanation": "Least exposure."
  },
  {
    "question": "Why use [ordered] for output records?",
    "options": [
      "Speed",
      "Predictable column order in reports and JSON",
      "Memory",
      "Law"
    ],
    "correctIndex": 1,
    "explanation": "Readable output."
  },
  {
    "question": "What's the main cost of $array += $item in a loop?",
    "options": [
      "Nothing",
      "Locks",
      "Each += allocates and copies a new array",
      "Sorting"
    ],
    "correctIndex": 2,
    "explanation": "Use List[T]."
  },
  {
    "question": "What is -Parallel good for in PowerShell 7?",
    "options": [
      "Debugging",
      "Ordering",
      "Running independent, slow tasks (I/O or remote) concurrently",
      "Tiny loops"
    ],
    "correctIndex": 2,
    "explanation": "Use ThrottleLimit."
  },
  {
    "question": "What belongs in a published module?",
    "options": [
      "Logs",
      "Secrets",
      "Manifest, tests, help, semantic version and signed or trusted source",
      "Only code"
    ],
    "correctIndex": 2,
    "explanation": "Gallery quality."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A log parser with reports and an alert

Implement ConvertFrom-LogLine (named-group regex, typed properties, skip bad lines with Write-Verbose), then use the pipeline to report counts by level, the slowest requests, the server error rate, per-service average latency and an alert flag.

**Starter code**

```powershell
$log = @"
2026-03-01 10:00:01 INFO  web  GET /home 200 35ms
2026-03-01 10:00:02 ERROR api  POST /pay 500 910ms
2026-03-01 10:00:03 INFO  web  GET /cart 200 48ms
2026-03-01 10:01:10 WARN  api  GET /items 200 700ms
2026-03-01 10:01:11 ERROR api  POST /pay 500 880ms
2026-03-01 10:02:00 INFO  web  GET /home 404 12ms
bad line that should be skipped
"@ -split "`n"

function ConvertFrom-LogLine {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][string]$Line)
    process {
        # match "<date> <time> <LEVEL> <service> <METHOD> <path> <status> <ms>ms" with named groups;
        # emit [pscustomobject] with Time ([datetime]), Level, Service, Path, Status ([int]), Ms ([int]);
        # for lines that do not match, Write-Verbose and emit nothing
    }
}

$entries = $log | ConvertFrom-LogLine
"parsed: $($entries.Count) of $($log.Count) lines"
"by level:"
$entries | Group-Object Level | Sort-Object Name | ForEach-Object { "  {0,-5} {1}" -f $_.Name, $_.Count }
"slowest:"
$entries | Sort-Object Ms -Descending | Select-Object -First 2 | ForEach-Object { "  {0} {1} {2}ms" -f $_.Service, $_.Path, $_.Ms }
$errRate = [math]::Round(100 * @($entries | Where-Object Status -ge 500).Count / $entries.Count, 1)
"server error rate: $errRate%"
$avg = $entries | Group-Object Service | ForEach-Object { [pscustomobject]@{ Service = $_.Name; AvgMs = [math]::Round(($_.Group | Measure-Object Ms -Average).Average, 1) } }
($avg | Sort-Object Service | ForEach-Object { "$($_.Service)=$($_.AvgMs)" }) -join "; "
$alert = $errRate -gt 10
"ALERT: $alert"
($entries | Select-Object Service, Path, Status | ConvertTo-Json -Compress -Depth 2).Length -gt 0
```

**Expected output**

```text
parsed: 6 of 7 lines
by level:
  ERROR 2
  INFO  3
  WARN  1
slowest:
  api /pay 910ms
  api /pay 880ms
server error rate: 33.3%
api=830; web=31.7
ALERT: True
True
```

<details><summary>Show solution and explanation</summary>

<pre><code>$log = @&quot;
2026-03-01 10:00:01 INFO  web  GET /home 200 35ms
2026-03-01 10:00:02 ERROR api  POST /pay 500 910ms
2026-03-01 10:00:03 INFO  web  GET /cart 200 48ms
2026-03-01 10:01:10 WARN  api  GET /items 200 700ms
2026-03-01 10:01:11 ERROR api  POST /pay 500 880ms
2026-03-01 10:02:00 INFO  web  GET /home 404 12ms
bad line that should be skipped
&quot;@ -split &quot;`n&quot;

function ConvertFrom-LogLine {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][string]$Line)
    process {
        if ($Line -match &#x27;^(?&lt;ts&gt;\S+ \S+) (?&lt;level&gt;[A-Z]+)\s+(?&lt;svc&gt;\w+)\s+(?&lt;method&gt;[A-Z]+) (?&lt;path&gt;\S+) (?&lt;status&gt;\d{3}) (?&lt;ms&gt;\d+)ms$&#x27;) {
            [pscustomobject]@{
                Time = [datetime]$Matches.ts; Level = $Matches.level; Service = $Matches.svc
                Path = $Matches.path; Status = [int]$Matches.status; Ms = [int]$Matches.ms
            }
        } else { Write-Verbose &quot;skipped: $Line&quot; }
    }
}

$entries = $log | ConvertFrom-LogLine
&quot;parsed: $($entries.Count) of $($log.Count) lines&quot;
&quot;by level:&quot;
$entries | Group-Object Level | Sort-Object Name | ForEach-Object { &quot;  {0,-5} {1}&quot; -f $_.Name, $_.Count }
&quot;slowest:&quot;
$entries | Sort-Object Ms -Descending | Select-Object -First 2 | ForEach-Object { &quot;  {0} {1} {2}ms&quot; -f $_.Service, $_.Path, $_.Ms }
$errRate = [math]::Round(100 * @($entries | Where-Object Status -ge 500).Count / $entries.Count, 1)
&quot;server error rate: $errRate%&quot;
$avg = $entries | Group-Object Service | ForEach-Object { [pscustomobject]@{ Service = $_.Name; AvgMs = [math]::Round(($_.Group | Measure-Object Ms -Average).Average, 1) } }
($avg | Sort-Object Service | ForEach-Object { &quot;$($_.Service)=$($_.AvgMs)&quot; }) -join &quot;; &quot;
$alert = $errRate -gt 10
&quot;ALERT: $alert&quot;
($entries | Select-Object Service, Path, Status | ConvertTo-Json -Compress -Depth 2).Length -gt 0
</code></pre>

<p>The function emits objects from process{} so the rest is ordinary pipeline work: Group-Object, Sort-Object, Measure-Object. Verified by running the solution with PowerShell 7.</p>
</details>

### A validating pipeline function and retry helper

Implement Test-Password (validation attributes, a List of problems, masked output) and Invoke-WithRetry (retry with attempt numbers, rethrow after the last attempt). Run both against the sample inputs.

**Starter code**

```powershell
function Test-Password {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][AllowEmptyString()][string]$Password,
          [ValidateRange(4,64)][int]$MinLength = 8)
    process {
        # collect problems: "too short", "no uppercase" (case-sensitive test), "no digit", "no symbol" (non-word, non-space char)
        # emit [pscustomobject] Password (masked with '*' of the same length), Valid (bool), Problems (comma-separated)
    }
}

function Invoke-WithRetry {
    param([scriptblock]$Action, [int]$Max = 3)
    # run $Action with the attempt number; on failure write "attempt N failed: <message>" and retry;
    # rethrow after the last attempt; return the action's result on success
}

'Passw0rd!', 'short1!', 'alllowercase1!', 'NoDigits!!', '' | Test-Password | ForEach-Object { "{0,-14} valid={1,-5} {2}" -f $_.Password, $_.Valid, $_.Problems }
try { Test-Password -Password 'x' -MinLength 2 } catch { "validation: rejected" }
$r = Invoke-WithRetry { param($n) if ($n -lt 3) { throw "flaky $n" } else { "ok on attempt $n" } }
$r
try { Invoke-WithRetry -Max 2 { throw "always" } } catch { "gave up: $($_.Exception.Message)" }
```

**Expected output**

```text
*********      valid=True
*******        valid=False too short, no uppercase
************** valid=False no uppercase
**********     valid=False no digit
               valid=False too short, no uppercase, no digit, no symbol
validation: rejected
attempt 1 failed: flaky 1
attempt 2 failed: flaky 2
ok on attempt 3
attempt 1 failed: always
attempt 2 failed: always
gave up: always
```

<details><summary>Show solution and explanation</summary>

<pre><code>function Test-Password {
    [CmdletBinding()]
    param([Parameter(Mandatory, ValueFromPipeline)][AllowEmptyString()][string]$Password,
          [ValidateRange(4,64)][int]$MinLength = 8)
    process {
        $problems = [System.Collections.Generic.List[string]]::new()
        if ($Password.Length -lt $MinLength) { $problems.Add(&quot;too short&quot;) }
        if ($Password -cnotmatch &#x27;[A-Z]&#x27;) { $problems.Add(&quot;no uppercase&quot;) }
        if ($Password -notmatch &#x27;\d&#x27;) { $problems.Add(&quot;no digit&quot;) }
        if ($Password -notmatch &#x27;[^\w\s]&#x27;) { $problems.Add(&quot;no symbol&quot;) }
        [pscustomobject]@{ Password = (&#x27;*&#x27; * $Password.Length); Valid = ($problems.Count -eq 0); Problems = ($problems -join &#x27;, &#x27;) }
    }
}

function Invoke-WithRetry {
    param([scriptblock]$Action, [int]$Max = 3)
    for ($i = 1; $i -le $Max; $i++) {
        try { return &amp; $Action $i }
        catch { Write-Output &quot;attempt $i failed: $($_.Exception.Message)&quot;; if ($i -eq $Max) { throw } }
    }
}

&#x27;Passw0rd!&#x27;, &#x27;short1!&#x27;, &#x27;alllowercase1!&#x27;, &#x27;NoDigits!!&#x27;, &#x27;&#x27; | Test-Password | ForEach-Object { &quot;{0,-14} valid={1,-5} {2}&quot; -f $_.Password, $_.Valid, $_.Problems }
try { Test-Password -Password &#x27;x&#x27; -MinLength 2 } catch { &quot;validation: rejected&quot; }
$r = Invoke-WithRetry { param($n) if ($n -lt 3) { throw &quot;flaky $n&quot; } else { &quot;ok on attempt $n&quot; } }
$r
try { Invoke-WithRetry -Max 2 { throw &quot;always&quot; } } catch { &quot;gave up: $($_.Exception.Message)&quot; }
</code></pre>

<p>Parameter validation rejects bad arguments before the body runs; process{} handles each piped value; try/catch with a loop gives retries and rethrows the final failure. Verified by running the solution with PowerShell 7.</p>
</details>
