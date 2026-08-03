# 06 · Testing with Pester

Every function you've written so far has been tested by eye — run it, look
at the output, decide if it's right. That doesn't scale, and it doesn't
protect you from a later change quietly breaking something you already
verified. **Pester** is PowerShell's testing framework: it lets you write
automated checks once and re-run all of them in seconds, forever.

## Installing Pester

```powershell
Install-Module -Name Pester -Force -Scope CurrentUser
Get-Module -ListAvailable Pester
```

Modern PowerShell (5.1+ on Windows, 7+ everywhere) usually ships with an
older built-in Pester version — `-Force -Scope CurrentUser` installs a
current one alongside it without needing admin rights.

## The module under test

```powershell
# Add.psm1
function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
}

function Get-Discount {
    param(
        [Parameter(Mandatory)]
        [double]$Price,
        [double]$PercentOff
    )
    if ($PercentOff -lt 0 -or $PercentOff -gt 100) {
        throw "PercentOff must be between 0 and 100"
    }
    return [math]::Round($Price * (1 - $PercentOff / 100), 2)
}

Export-ModuleMember -Function Add-Numbers, Get-Discount
```

## Your first test file

Pester test files follow the naming convention `<Name>.Tests.ps1`, live next
to the code they test, and are built from `Describe`, `Context`, and `It`
blocks.

```powershell
# Add.Tests.ps1
BeforeAll {
    Import-Module "$PSScriptRoot/Add.psm1" -Force
}

Describe "Add-Numbers" {
    It "adds two positive numbers" {
        Add-Numbers -A 2 -B 3 | Should -Be 5
    }

    It "handles negative numbers" {
        Add-Numbers -A -5 -B 10 | Should -Be 5
    }

    Context "with zero" {
        It "adding zero returns the same number" {
            Add-Numbers -A 7 -B 0 | Should -Be 7
        }
    }
}

Describe "Get-Discount" {
    It "applies a percentage discount" {
        Get-Discount -Price 100 -PercentOff 20 | Should -Be 80
    }

    It "throws for an out-of-range percent" {
        { Get-Discount -Price 100 -PercentOff 150 } | Should -Throw
    }

    It "returns a value of type double" {
        Get-Discount -Price 50 -PercentOff 10 | Should -BeOfType [double]
    }
}
```

```powershell
Invoke-Pester -Path ./Add.Tests.ps1 -Output Detailed
```

```text
Describing Add-Numbers
  [+] adds two positive numbers 4ms
  [+] handles negative numbers 2ms
  Context with zero
    [+] adding zero returns the same number 1ms
Describing Get-Discount
  [+] applies a percentage discount 3ms
  [+] throws for an out-of-range percent 2ms
  [+] returns a value of type double 1ms
Tests Passed: 6, Failed: 0, Skipped: 0, Inconclusive: 0, NotRun: 0
```

- `Describe` groups all tests for one thing (usually one function).
- `Context` sub-groups related scenarios within a `Describe` (optional).
- `It` is a single test case — the string describes the expected behavior
  in plain English, which doubles as living documentation.
- `BeforeAll` runs once before any test in its scope — the natural place to
  `Import-Module` the code under test.

## `Should`: Pester's assertion syntax

```powershell
5              | Should -Be 5
"hello"        | Should -Be "hello"
$true          | Should -BeTrue
$null          | Should -BeNullOrEmpty
@(1, 2, 3)     | Should -Contain 2
{ 1 / 0 }      | Should -Throw
10             | Should -BeGreaterThan 5
"abc"          | Should -Match "^a"
```

`Should` reads left-to-right almost like English, which is intentional —
tests are meant to be readable by someone who didn't write them. Note the
`{ 1 / 0 }` for `-Throw`: the code under test has to be wrapped in a script
block, otherwise `1 / 0` would throw *before* Pester ever got a chance to
catch it as part of the assertion.

## The trap: testing implementation instead of behavior

```powershell
# Fragile: breaks if you ever rename an internal variable, even if behavior is unchanged
It "uses a variable called total" {
    (Get-Content Add.psm1) -match '\$total' | Should -Not -BeNullOrEmpty
}

# Robust: tests what the function actually promises to callers
It "adds two numbers correctly" {
    Add-Numbers -A 2 -B 3 | Should -Be 5
}
```

Good tests describe **what** a function does (its inputs and outputs), not
**how** it does it internally. A refactor that changes internals but keeps
behavior identical should never break a well-written test — if it does,
the test was checking the wrong thing.

## Mocking: isolating the function under test

```powershell
Describe "Get-WeatherSummary" {
    BeforeAll {
        function Get-WeatherSummary {
            param([string]$City)
            $data = Invoke-RestMethod -Uri "https://api.example.com/weather?city=$City"
            return "It is $($data.TempC) degrees in $City"
        }
    }

    It "formats the API response correctly" {
        Mock Invoke-RestMethod { return @{ TempC = 22 } }

        Get-WeatherSummary -City "Chennai" | Should -Be "It is 22 degrees in Chennai"

        Should -Invoke Invoke-RestMethod -Times 1 -Exactly
    }
}
```

`Mock` replaces a real command (here, `Invoke-RestMethod`) with a fake
version for the duration of the test — so the test doesn't depend on a real
network call, real API key, or real weather. `Should -Invoke ... -Times`
additionally verifies the mocked command was actually called the expected
number of times, catching bugs where a function silently skips the call it
was supposed to make.

Note that the function-under-test is defined inside `BeforeAll`, not as
bare code at the top of the file. Pester runs a test file in two phases —
**Discovery** (scanning for `Describe`/`Context`/`It` blocks) and **Run**
(actually executing them) — and code sitting outside any block only
reliably executes during Discovery. A function defined that way can end up
invisible by the time `It` blocks actually run; defining it inside
`BeforeAll` (or importing a real module there, as in the first example)
guarantees it exists for the Run phase.

## `BeforeEach` / `AfterEach`: fresh state per test

```powershell
Describe "Counter behavior" {
    BeforeEach {
        $script:counter = 0
    }

    It "starts at zero" {
        $script:counter | Should -Be 0
    }

    It "increments independently of other tests" {
        $script:counter++
        $script:counter | Should -Be 1
    }
}
```

`BeforeEach` runs before **every** `It` block in its scope (not just once
like `BeforeAll`) — essential when tests need to start from a clean,
predictable state instead of accidentally depending on leftover state from
a previous test running first.

## Data-driven tests with `-ForEach`

```powershell
Describe "Get-Discount with multiple inputs" {
    It "gives <Expected> for <Price> at <Percent>% off" -ForEach @(
        @{ Price = 100; Percent = 10; Expected = 90 }
        @{ Price = 200; Percent = 50; Expected = 100 }
        @{ Price = 50;  Percent = 0;  Expected = 50 }
    ) {
        Get-Discount -Price $Price -PercentOff $Percent | Should -Be $Expected
    }
}
```

(Assuming `Get-Discount` is already available via a `BeforeAll` /
`Import-Module`, as shown earlier.)

`-ForEach` runs the same `It` block once per hashtable in the array,
substituting each key as a variable inside the block — a compact way to
cover many input/output combinations without copy-pasting nearly-identical
tests.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `Describe` | groups all tests for one unit (usually a function) |
| `Context` | sub-groups related scenarios inside a `Describe` |
| `It` | a single test case, named in plain English |
| `Should -Be`, `-BeTrue`, `-Throw`, `-Contain`, `-Match` | common assertions |
| `BeforeAll` / `AfterAll` | run once per `Describe`, before/after all tests |
| `BeforeEach` / `AfterEach` | run before/after **every** `It` |
| `Mock` | replace a real command with a fake for the test |
| `Should -Invoke ... -Times` | verify a mocked command was called N times |
| `-ForEach @(...)` | run one `It` block once per data row |
| `Invoke-Pester -Path ... -Output Detailed` | run tests from the command line |

## Exercise

Add a function `Test-PasswordStrength` to a module that returns `$true` if
a password is at least 8 characters and contains at least one digit,
`$false` otherwise. Write a Pester test file for it that uses `-ForEach`
to check at least four cases (too short, no digit, valid, and a boundary
case at exactly 8 characters), and confirm all tests pass with
`Invoke-Pester`.
