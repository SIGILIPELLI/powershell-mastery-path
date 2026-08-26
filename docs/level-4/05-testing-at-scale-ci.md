# 05 · Testing at Scale & CI

A handful of Pester tests running with `Invoke-Pester -Output Detailed`
is fine at your desk. In a CI pipeline running hundreds of tests across
many files, you need three more things: selective execution (tags), a
machine-readable result format CI systems can parse, and code coverage —
a number telling you how much of the code your tests actually exercise.

## Tagging tests for selective runs

```powershell
Describe "Get-Factorial" -Tag "Unit" {
    It "computes <N>! = <Expected>" -ForEach @(
        @{ N = 0; Expected = 1 }
        @{ N = 1; Expected = 1 }
        @{ N = 5; Expected = 120 }
        @{ N = 10; Expected = 3628800 }
    ) {
        Get-Factorial -N $N | Should -Be $Expected
    }

    It "throws for negative input" -Tag "EdgeCase" {
        { Get-Factorial -N -1 } | Should -Throw
    }
}

Describe "Get-Factorial performance" -Tag "Slow" {
    It "completes 15! in reasonable time" {
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        Get-Factorial -N 15 | Should -Be 1307674368000
        $sw.Stop()
        $sw.ElapsedMilliseconds | Should -BeLessThan 500
    }
}
```

`-Tag` on `Describe` or `It` lets a CI pipeline run subsets deliberately —
fast unit tests on every push, slower integration/performance tests only
on a nightly schedule or before a release, rather than paying the full
suite's runtime on every single commit.

## Filtering by tag and generating machine-readable output

```powershell
$config = New-PesterConfiguration
$config.Run.Path = './Math.Tests.ps1'
$config.Filter.Tag = 'Unit'
$config.Output.Verbosity = 'Detailed'
$config.TestResult.Enabled = $true
$config.TestResult.OutputPath = './testresults.xml'
$config.TestResult.OutputFormat = 'NUnitXml'
$config.CodeCoverage.Enabled = $true
$config.CodeCoverage.Path = './Math.psm1'
$config.CodeCoverage.OutputPath = './coverage.xml'

Invoke-Pester -Configuration $config
```

```text
Pester v6.0.1
Filter 'Tag' set to ('Unit').
Filters selected 5 tests to run.
Starting code coverage.
Describing Get-Factorial
  [+] computes 0! = 1 67ms
  [+] computes 1! = 1 6ms
  [+] computes 5! = 120 8ms
  [+] computes 10! = 3628800 6ms
  [+] throws for negative input 33ms
Tests Passed: 5, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 1
Processing code coverage result.
Covered 100% / 75%. 9 analyzed Commands in 1 File.
```

`Filters selected 5 tests to run` confirms the `-Tag Unit` filter worked —
the `Slow`-tagged performance test was skipped entirely (it shows as
`NotRun: 1`), exactly the selective-execution behavior CI needs.

`OutputFormat 'NUnitXml'` writes results in a format essentially every CI
system (GitHub Actions, Azure DevOps, Jenkins) knows how to parse into a
readable test report with pass/fail annotations directly on the pipeline
run — far more useful for a team than scrolling raw console output.

## Reading a code coverage result

```text
Covered 100% / 75%. 9 analyzed Commands in 1 File.
```

Pester reports coverage as **command coverage** (which individual
PowerShell statements/commands actually executed during the test run),
not line coverage — the "100%" here is the analyzed-file's overall figure
in this specific example, and the "75%" is Pester's coverage threshold
(configurable via `$config.CodeCoverage.CoveragePercentTarget`) it's
comparing against; a run falling under the target can be made to fail the
build the same way a failed assertion does.

```powershell
$config.CodeCoverage.CoveragePercentTarget = 80
```

## The trap: high coverage numbers hide untested *behavior*

A recursive function like `Get-Factorial` can show 100% command coverage
from tests that only ever call it with small, well-behaved inputs — every
line executes, but nothing tests what happens with `-N 0` as a boundary,
deeply negative input, or a value large enough to overflow `[int]`.
Coverage percentage measures **which lines ran**, not **which scenarios
were verified** — a genuinely well-tested function needs both a coverage
number *and* deliberate boundary/edge-case `It` blocks like the
"negative input" test above, not just enough calls to touch every line
once.

## Structuring a large suite for speed

For a real module with dozens of test files, two habits keep CI fast as
the suite grows:

- **Split by tag, not by guessing** — `Unit` tests (no I/O, no sleep,
  milliseconds each) run on every push; `Integration`/`Slow` tests
  (hitting real services, deliberately timed) run on a schedule or before
  merge to `main` only.
- **Parallelize test files, not individual `It` blocks** — Pester itself
  runs sequentially within one invocation; splitting test files across
  multiple `ForEach-Object -Parallel` (module 01) or across separate CI
  matrix jobs is how a large suite's wall-clock time stays reasonable as
  it grows past a few hundred tests.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `-Tag "Name"` on `Describe`/`It` | group tests for selective execution |
| `$config.Filter.Tag = 'Unit'` | run only tests matching a tag |
| `$config.TestResult.OutputFormat = 'NUnitXml'` | CI-parseable results |
| `$config.CodeCoverage.Enabled = $true` | measure which commands executed |
| `CoveragePercentTarget` | fail the build if coverage drops below a threshold |
| Command coverage ≠ scenario coverage | 100% coverage can still miss edge cases |
| Split suites by tag (`Unit` vs `Slow`/`Integration`) | keep fast feedback fast |

## Exercise

Take the `AdminToolkit` module from Level 3's project, tag its existing
tests `Unit`, add at least two new `Integration`-tagged tests that
actually hit real `Get-PSDrive`/`Get-Process` (no mocking) and assert
only on shape (correct properties present, no exceptions) rather than
exact values, then run the suite twice with `New-PesterConfiguration` —
once filtered to `Unit` only, once unfiltered — and generate a coverage
report for each, comparing the coverage percentage between the two runs.
