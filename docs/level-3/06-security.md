# 06 · Security

Automation scripts routinely need secrets — API keys, service account
passwords, connection strings — which makes them a common place for those
secrets to leak: hardcoded in source, printed to a log, or sitting as
plaintext in a variable that ends up in an error message or transcript.
This module covers PowerShell's tools for handling secrets more carefully,
plus the security controls people commonly *over*-trust.

## `SecureString` and `PSCredential`

```powershell
$secure = ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force
$cred = [PSCredential]::new("svc-account", $secure)

$cred.UserName
$cred.Password.GetType().Name
```

```text
svc-account
SecureString
```

`SecureString` keeps the characters encrypted in memory rather than as a
plain managed string, and — critically — most cmdlets that accept
credentials print `System.Security.SecureString` instead of the actual
value if you accidentally output it. It's damage limitation, not
unbreakable encryption: anyone with debugger-level access to the process
can still recover it, and .NET has actually deprecated `SecureString` in
newer runtimes in favor of secret-vault patterns — but for scripts that
still need to pass a credential around, it beats a bare string.

```powershell
$plain = $cred.GetNetworkCredential().Password
"Length: $($plain.Length)"
```

```text
Length: 11
```

`GetNetworkCredential().Password` is the sanctioned way to get the
plaintext back out — decrypt at the last possible moment, right before
handing it to whatever actually needs it (an API call, a connection
string), and let the plaintext variable go out of scope immediately after.

## `ConvertTo-SecureString -AsPlainText -Force` is a smell, not a solution

Note that in the example above, we *started* from plaintext to build the
demo — that line is exactly what you should almost never write in real
code, because the plaintext already exists as a literal in your source by
the time you convert it. `ConvertTo-SecureString` is meant to receive an
*already-secure* string (from `Read-Host -AsSecureString`, or decrypted
from a secret store), not to wrap a hardcoded password. If you see
`-AsPlainText -Force` with a literal string next to it in a script, that's
a hardcoded secret with a `SecureString` costume on.

## The trap: a `[string]` secret parameter leaks everywhere strings do

```powershell
function Connect-Service {
    param(
        [Parameter(Mandatory)]
        [string]$ApiKey
    )
    "Connecting with key ending in ...$($ApiKey.Substring($ApiKey.Length - 4))"
}
Connect-Service -ApiKey "sk-super-secret-123456"
```

```text
Connecting with key ending in ...3456
```

That function *works*, but `$ApiKey` as a plain string will show up in
full in `Get-History`, in a PowerShell transcript (`Start-Transcript`), in
verbose tracing, and in any error message that happens to interpolate it.
Typing the parameter as `[SecureString]` instead closes most of those:

```powershell
function Connect-ServiceSecure {
    param(
        [Parameter(Mandatory)]
        [SecureString]$ApiKey
    )
    $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($ApiKey)
    try {
        $plain = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($bstr)
        "Connecting with key ending in ...$($plain.Substring($plain.Length - 4))"
    } finally {
        [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
    }
}

$secureKey = ConvertTo-SecureString "sk-super-secret-123456" -AsPlainText -Force
Connect-ServiceSecure -ApiKey $secureKey
```

```text
Connecting with key ending in ...3456
```

The `finally` block matters: `ZeroFreeBSTR` explicitly zeroes the
unmanaged memory holding the decrypted value the moment you're done with
it, instead of leaving it sitting in memory until garbage collection gets
around to it.

## `Get-FileHash`: integrity checks, not password storage

```powershell
"hello world" | Out-File ./demo.txt
(Get-FileHash -Path ./demo.txt -Algorithm SHA256).Hash
```

```text
A948904F2F0F479B8F8197694B30184B0D2ED1C1CD2A1EC0FB85D299A192A447
```

`Get-FileHash` is for verifying a downloaded file or script hasn't been
tampered with (compare against a published hash) — it is **not** a
password-hashing function. Never build "secure" password storage on top
of `Get-FileHash`/`SHA256` directly; password hashing needs a slow,
salted algorithm (bcrypt/PBKDF2/Argon2), which is outside PowerShell's
built-in toolset and belongs in a dedicated identity system, not a script.

## The trap: `Select-String` finds hardcoded secrets you forgot about

```powershell
"`$password = 'hardcoded-oops'" | Out-File ./bad-script.ps1
Select-String -Path ./bad-script.ps1 -Pattern 'password\s*='
```

```text
bad-script.ps1:1:$password = 'hardcoded-oops'
```

Running a quick `Select-String -Pattern 'password|apikey|secret|token\s*='`
sweep across a script repo before committing is a cheap habit that catches
the single most common real-world leak: a secret typed directly into
source during testing and never removed.

## Execution policy is a guardrail against mistakes, not an attacker

```powershell
Get-ExecutionPolicy -List
```

```text
        Scope ExecutionPolicy
        ----- ---------------
MachinePolicy    Unrestricted
   UserPolicy    Unrestricted
      Process    Unrestricted
  CurrentUser    Unrestricted
 LocalMachine    Unrestricted
```

`ExecutionPolicy` only gates whether a script *runs* by double-click or
default invocation — it is trivially bypassed (`powershell -ExecutionPolicy
Bypass -File script.ps1`, or piping content through `Invoke-Expression`)
and Microsoft documents it explicitly as **not a security boundary**. Its
real purpose is preventing accidental execution (stopping a downloaded
`.ps1` from running just because you double-clicked it), not stopping a
determined attacker. Don't design a security model that depends on it.

## Cheat sheet

| Concept | Use |
|---|---|
| `SecureString` | encrypted-in-memory string; damage limitation, not unbreakable |
| `[PSCredential]::new($user, $secureString)` | pair a username with a secure password |
| `$cred.GetNetworkCredential().Password` | decrypt only at the point of use |
| `ConvertTo-SecureString -AsPlainText -Force` on a literal | smell — hardcoded secret in disguise |
| `[Runtime.InteropServices.Marshal]::ZeroFreeBSTR` | explicitly clear decrypted memory after use |
| `Get-FileHash -Algorithm SHA256` | integrity checks — never password storage |
| `Select-String -Pattern 'password\|apikey\|secret'` | quick pre-commit secret sweep |
| `ExecutionPolicy` | accident prevention, not an attacker-facing security boundary |

## Exercise

Write a function `Test-SecretExposure` that scans a directory of `.ps1`
files with `Select-String` for common secret patterns (`password\s*=`,
`apikey\s*=`, `-AsPlainText`) and returns a report object per match
(file, line number, matched text). Run it against a folder containing at
least one deliberately planted "leak" and confirm it's caught, then run it
against a version using `SecureString`/`Read-Host -AsSecureString`
properly and confirm that one is clean.
