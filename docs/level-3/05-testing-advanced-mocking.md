# 05 · Testing Advanced (Pester Mocking)

Level 2's Pester module covered `Mock` for a function calling an external
command directly. Real modules mock deeper: a public function calling a
*sibling function in the same module*, filtering mocks by argument, and
asserting a mock was called with specific parameters — not just "called."

## The module under test

```powershell
# Deploy.psm1
function Get-ServiceHealth {
    param([string]$Uri)
    $resp = Invoke-RestMethod -Uri $Uri -ErrorAction Stop
    if ($resp.status -eq 'ok') { return $true }
    return $false
}

function Publish-Release {
    param(
        [Parameter(Mandatory)][string]$Version,
        [Parameter(Mandatory)][string]$HealthUri
    )
    if (-not (Test-Path Env:\DEPLOY_TOKEN)) {
        throw "DEPLOY_TOKEN not set"
    }
    if (-not (Get-ServiceHealth -Uri $HealthUri)) {
        Write-Warning "Service unhealthy before deploy, aborting"
        return $false
    }
    Copy-Item -Path "./build/*" -Destination "/releases/$Version" -Recurse -Force
    Write-Output "Deployed $Version"
    return $true
}

Export-ModuleMember -Function Get-ServiceHealth, Publish-Release
```

`Publish-Release` calls `Get-ServiceHealth` — a function from the *same*
module — before doing anything destructive. Testing `Publish-Release` in
isolation means faking that call too, not just the outer HTTP request.

## The trap: mocking a function doesn't work without `-ModuleName`

The first, very natural attempt:

```powershell
Describe "Publish-Release" {
    It "deploys when the health check passes" {
        Mock Get-ServiceHealth { $true }
        Mock Copy-Item {}

        Publish-Release -Version "1.2.3" -HealthUri "https://x/health" | Should -BeTrue
    }
}
```

```text
[-] deploys when the health check passes 258ms
 SocketException: nodename nor servname provided, or not known
 HttpRequestException: nodename nor servname provided, or not known (x:443)
 at Get-ServiceHealth, Deploy.psm1:3
 at Publish-Release, Deploy.psm1:16
```

The mock is defined in the **test file's** scope, but `Publish-Release`
calls `Get-ServiceHealth` from *inside the module's own scope* — and
those are different scopes for command resolution purposes. The mock
never intercepts the internal call, so the real `Invoke-RestMethod`
fires and fails against a fake hostname. The fix is `-ModuleName`, which
installs the mock into the module's scope instead of the test's:

```powershell
Mock Get-ServiceHealth -ModuleName Deploy { $true }
Mock Copy-Item -ModuleName Deploy {}
```

Every `Mock` and every `Should -Invoke` targeting a function called
*from inside the module under test* needs the same `-ModuleName` — mixing
a `-ModuleName` mock with a plain `Should -Invoke` (or vice versa)
produces `Could not find Mock for command ... in script scope`, because
Pester is looking for the assertion in a different scope than where the
mock actually lives.

## The full test file

```powershell
BeforeAll {
    Import-Module "$PSScriptRoot/Deploy.psm1" -Force
    $env:DEPLOY_TOKEN = "test-token"
}

AfterAll {
    Remove-Item Env:\DEPLOY_TOKEN -ErrorAction SilentlyContinue
}

Describe "Publish-Release" {
    BeforeEach {
        Mock Copy-Item {} -ModuleName Deploy
        Mock Write-Output {} -ModuleName Deploy
    }

    It "deploys when the health check passes" {
        Mock Get-ServiceHealth -ModuleName Deploy { $true }

        $result = Publish-Release -Version "1.2.3" -HealthUri "https://x/health"

        $result | Should -BeTrue
        Should -Invoke Copy-Item -ModuleName Deploy -Times 1 -Exactly
    }

    It "aborts and does not copy when the health check fails" {
        Mock Get-ServiceHealth -ModuleName Deploy { $false }

        $result = Publish-Release -Version "1.2.3" -HealthUri "https://x/health"

        $result | Should -BeFalse
        Should -Invoke Copy-Item -ModuleName Deploy -Times 0 -Exactly
    }

    It "passes the health URI through unchanged" {
        Mock Get-ServiceHealth -ModuleName Deploy { $true } `
            -ParameterFilter { $Uri -eq "https://specific/health" }

        Publish-Release -Version "9.9.9" -HealthUri "https://specific/health" | Out-Null

        Should -Invoke Get-ServiceHealth -ModuleName Deploy -Times 1 -Exactly -ParameterFilter {
            $Uri -eq "https://specific/health"
        }
    }

    It "throws when DEPLOY_TOKEN is missing" {
        Remove-Item Env:\DEPLOY_TOKEN
        { Publish-Release -Version "1.0.0" -HealthUri "https://x/health" } |
            Should -Throw "*DEPLOY_TOKEN*"
        $env:DEPLOY_TOKEN = "test-token"
    }
}

Describe "Get-ServiceHealth real call, mocked at the boundary" {
    It "returns true when the API reports ok" {
        Mock Invoke-RestMethod -ModuleName Deploy { [pscustomobject]@{ status = 'ok' } }
        Get-ServiceHealth -Uri "https://x/health" | Should -BeTrue
    }

    It "returns false when the API reports degraded" {
        Mock Invoke-RestMethod -ModuleName Deploy { [pscustomobject]@{ status = 'degraded' } }
        Get-ServiceHealth -Uri "https://x/health" | Should -BeFalse
    }
}
```

```powershell
Invoke-Pester -Path ./Deploy.Tests.ps1 -Output Detailed
```

```text
Describing Publish-Release
  [+] deploys when the health check passes 270ms
WARNING: Service unhealthy before deploy, aborting
  [+] aborts and does not copy when the health check fails 16ms
  [+] passes the health URI through unchanged 37ms
  [+] throws when DEPLOY_TOKEN is missing 61ms
Describing Get-ServiceHealth real call, mocked at the boundary
  [+] returns true when the API reports ok 24ms
  [+] returns false when the API reports degraded 13ms
Tests Passed: 6, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
```

The `WARNING:` line printing during a passing test is expected — the test
that exercises the "unhealthy, abort" branch legitimately triggers
`Write-Warning`, and Pester doesn't suppress warnings by default.

## `-ParameterFilter`: mocking (or asserting) conditionally

`-ParameterFilter { $Uri -eq "https://specific/health" }` on a `Mock`
means that fake return value only applies when the filter matches — a
call with a different `$Uri` falls through to any other matching mock, or
the real command if none match. The same filter on `Should -Invoke` scopes
the assertion to only count calls matching it, which is how the third test
above proves the exact URI, not just *some* URI, was passed through.

## `Should -Invoke -Times 0`: proving something did NOT happen

Testing that a mocked side-effecting command (`Copy-Item`, an email send,
a database write) was **skipped** on a failure path is just as important
as testing it ran on the happy path — `-Times 0 -Exactly` makes that an
explicit, checked assertion instead of an assumption that no error means
no unwanted copy happened.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `Mock Cmd -ModuleName X` | mock a command as seen from inside module `X` |
| `Should -Invoke Cmd -ModuleName X` | assert against a mock defined with the same `-ModuleName` |
| `-ParameterFilter { ... }` | mock/assert conditionally, based on the arguments passed |
| `Should -Invoke ... -Times 0 -Exactly` | prove a side effect did NOT happen |
| `Should -Invoke ... -Times N -Exactly` | prove a call happened exactly N times, no more |
| `BeforeEach { Mock ... }` | fresh, reset mocks for every `It` |
| `-Throw "*substring*"` | assert an exception message contains specific text |

## Exercise

Extend `Deploy.psm1` with a `Send-DeployNotification` function that
`Publish-Release` calls after a successful deploy (mock it out in the
existing passing tests so they don't break). Write a new `It` block that
mocks it with `-ModuleName Deploy` and asserts, with `-ParameterFilter`,
that it was called with the correct `$Version` — then a second `It`
confirming it is **not** called when the health check fails.
