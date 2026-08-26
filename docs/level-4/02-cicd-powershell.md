# 02 · CI/CD with PowerShell

A module isn't production-ready because it works on your machine — it's
ready when a pipeline can build it, test it, and fail loudly the moment
something breaks, without a human watching. This module writes that
pipeline: a real build script driving real Pester, then wires it into
GitHub Actions.

## A build script as the single entry point

```powershell
# build.ps1
[CmdletBinding()]
param(
    [ValidateSet('Build','Test','All')]
    [string]$Task = 'All'
)

function Invoke-Build {
    Write-Output "==> Building module..."
    if (-not (Test-Path ./out)) { New-Item -ItemType Directory -Path ./out | Out-Null }
    Copy-Item ./src/*.psm1 ./out -Force
    Write-Output "Build complete."
}

function Invoke-Tests {
    Write-Output "==> Running Pester tests..."
    $config = New-PesterConfiguration
    $config.Run.Path = './tests'
    $config.Run.Exit = $true
    $config.Output.Verbosity = 'Detailed'
    $config.Run.PassThru = $true
    $result = Invoke-Pester -Configuration $config
    if ($result.FailedCount -gt 0) {
        throw "$($result.FailedCount) test(s) failed"
    }
    Write-Output "All $($result.PassedCount) tests passed."
}

switch ($Task) {
    'Build' { Invoke-Build }
    'Test'  { Invoke-Tests }
    'All'   { Invoke-Build; Invoke-Tests }
}
```

```text
==> Building module...
Build complete.
==> Running Pester tests...
Pester v6.0.1
Describing Add-Two
  [+] adds 100ms
Tests Passed: 1, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
All 1 tests passed.
```

This is the whole point of a build script: `Build`, `Test`, and `All` are
callable the same way whether a human runs `./build.ps1 -Task Test` at
their desk or CI runs the exact same command. `New-PesterConfiguration`
(Pester 5+'s configuration object) is preferred over the older positional
`Invoke-Pester -Path ... -PassThru` form because it makes every setting —
verbosity, exit behavior, which paths to run — explicit and discoverable.

## The trap: a passing test run that still exits 0 on failure

```powershell
Invoke-Pester -Path ./tests   # naive version
Write-Output "Build succeeded"   # this line runs regardless of test results!
```

Without checking `$result.FailedCount` (or setting `$config.Run.Exit =
$true`, which makes Pester itself set the process exit code), a script
that just calls `Invoke-Pester` and moves on will report success to CI
even when tests failed — because `Invoke-Pester` by default doesn't throw
or exit nonzero on its own. The `if ($result.FailedCount -gt 0) { throw }`
line above is what actually connects test failure to pipeline failure;
skip it and you get a green checkmark on a red build.

## GitHub Actions workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Pester
        shell: pwsh
        run: |
          Install-Module -Name Pester -Force -SkipPublisherCheck -Scope CurrentUser

      - name: Run build and tests
        shell: pwsh
        run: ./build.ps1 -Task All
```

`runs-on: ubuntu-latest` matters here specifically because PowerShell 7 is
cross-platform — there's no reason to pay for a Windows runner unless the
module actually needs Windows-only cmdlets (like `ActiveDirectory` from
module 08 of Level 3). `shell: pwsh` tells Actions to interpret each
`run:` block with PowerShell instead of the default bash.

## The trap: CI's PowerShell isn't your PowerShell

A script that runs cleanly on your machine can fail in CI for reasons
that have nothing to do with the code:

- **No installed modules** — CI runners start clean; `Install-Module
  Pester` (or any dependency) has to be an explicit pipeline step, not
  assumed to already be there.
- **No profile** — CI never loads `$PROFILE`; anything your interactive
  session quietly set up (an alias, a default parameter value via
  `$PSDefaultParameterValues`) won't exist.
- **Case sensitivity** — Linux runners (the common CI default) have a
  case-sensitive filesystem; `Import-Module ./Src/Calc.psm1` failing to
  find `./src/Calc.psm1` on Linux CI while working fine on Windows/macOS
  is one of the most common cross-platform CI surprises.

## Exit codes: what CI actually checks

```powershell
try {
    ./build.ps1 -Task All
} catch {
    Write-Error $_
    exit 1
}
exit 0
```

CI systems don't parse your script's console output to decide pass/fail —
they check the process exit code. `$config.Run.Exit = $true` in the
Pester config makes `Invoke-Pester` itself exit with a nonzero code on
failure, which is usually enough; wrapping the whole build in a
try/catch that explicitly calls `exit 1` on any thrown error is the extra
layer of insurance for anything *outside* Pester (a build step failing,
for instance) that also needs to fail the pipeline correctly.

## Cheat sheet

| Piece | Purpose |
|---|---|
| `build.ps1 -Task Build\|Test\|All` | one entry point, same on a laptop and in CI |
| `New-PesterConfiguration` | explicit, discoverable test-run settings |
| `$config.Run.Exit = $true` | makes Pester set the process exit code on failure |
| Checking `$result.FailedCount` | connects test failure to pipeline failure |
| `shell: pwsh` in GitHub Actions | run YAML `run:` blocks as PowerShell |
| `runs-on: ubuntu-latest` | cheaper, cross-platform-safe unless Windows-only cmdlets are needed |
| Case-sensitive paths on Linux CI | common source of "works on my machine" |
| `exit 1` on any caught error | explicit pipeline failure signal beyond Pester's own |

## Exercise

Add a `Lint` task to `build.ps1` that runs `Invoke-ScriptAnalyzer`
(PSScriptAnalyzer) against `./src` and fails the build (throws, causing a
nonzero exit) on any `Error`-severity finding. Wire it into the GitHub
Actions workflow as a separate step before `Test`, and confirm — by
deliberately introducing an obvious lint issue (like an unused variable)
— that the workflow fails at the lint step rather than the test step.
