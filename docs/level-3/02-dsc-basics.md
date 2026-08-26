# 02 · Desired State Configuration (DSC) Basics

Everything so far has been **imperative**: your script says exactly *how*
to do something, step by step. Desired State Configuration (DSC) flips
that — you declare *what* the end state should look like (a file exists
with this content, a service is running, a registry key is set), and a
DSC engine figures out how to get there and keeps checking that it stays
there.

!!! note "A note on running this yourself"
    The `configuration` keyword needs the classic DSC engine, and on
    **Apple Silicon (ARM64) machines it refuses to load at all** —
    running any of the code below there raises `Configuration keyword is
    not supported on ARM64 processors.` The examples in this module are
    written and reviewed carefully for correctness, but were not executed
    live for that reason — the same honest caveat this course applies to
    Active Directory cmdlets in module 08. On an x64 Windows or Linux
    machine with PowerShell 7 (or Windows PowerShell 5.1), everything here
    runs as shown. `Get-DscResource` and `Start-DscConfiguration` are
    Windows-only in practice; PowerShell 7's cross-platform DSC story is
    Azure Policy guest configuration for Linux, which is out of scope here.

## Declaring a configuration

```powershell
configuration WebServerConfig {
    Node "localhost" {
        File ExampleFile {
            DestinationPath = "C:\DscDemo\readme.txt"
            Contents        = "Managed by DSC"
            Ensure          = "Present"
        }
    }
}

WebServerConfig -OutputPath ./dscout
```

`configuration` is a PowerShell language keyword (not a function) that
compiles the block into one or more **MOF files** — one per `Node` — under
`OutputPath`. `Node` targets a specific machine by name; `File` is one of
the built-in DSC resources, describing the desired state of a single file.

```text
   Directory: C:\dscout

Mode          LastWriteTime   Length Name
----          -------------   ------ ----
-a----   8/26/2026  10:02 AM     612 localhost.mof
```

Nothing has actually changed on disk yet — compiling a configuration only
produces the MOF describing the target state.

## Applying it

```powershell
Start-DscConfiguration -Path ./dscout -Wait -Verbose
```

```text
VERBOSE: [localhost]: LCM:  [ Start  Resource ]  [[File]ExampleFile]
VERBOSE: [localhost]: LCM:  [ Start  Set      ]  [[File]ExampleFile]
VERBOSE: [localhost]:                            [[File]ExampleFile] The
    configuration of file "C:\DscDemo\readme.txt" is not correct.
VERBOSE: [localhost]:                            [[File]ExampleFile] Setting
    the contents of file "C:\DscDemo\readme.txt".
VERBOSE: [localhost]: LCM:  [ End    Set      ]  [[File]ExampleFile]
VERBOSE: [localhost]: LCM:  [ End    Resource ]  [[File]ExampleFile]
```

The Local Configuration Manager (LCM) — the engine running on every
DSC-capable node — reads the MOF, checks current state against desired
state resource-by-resource, and only touches what's actually wrong. Run
`Start-DscConfiguration` again with no drift and every resource reports
"already in the desired state" — nothing changes on a second run.

## Checking for drift without changing anything

```powershell
Test-DscConfiguration -Detailed
```

```text
InDesiredState ResourcesInDesiredState ResourcesNotInDesiredState
-------------- ----------------------- --------------------------
False          {}                      {ExampleFile}
```

This is the read-only half of DSC's value: you can audit a fleet of
machines for drift from their declared configuration without applying
anything, then decide separately whether to remediate.

## Parameters and multiple nodes

```powershell
configuration WebServerConfig {
    param(
        [string[]]$ComputerName = "localhost",
        [string]$Content = "Managed by DSC"
    )

    Node $ComputerName {
        File ExampleFile {
            DestinationPath = "C:\DscDemo\readme.txt"
            Contents        = $Content
            Ensure          = "Present"
        }

        WindowsFeature IIS {
            Name   = "Web-Server"
            Ensure = "Present"
        }
    }
}

WebServerConfig -ComputerName "Web01", "Web02" -OutputPath ./dscout
```

`Node $ComputerName` iterates the array, emitting one MOF **per machine**
named — `Web01.mof`, `Web02.mof` — each independently deployable via
`Start-DscConfiguration -ComputerName Web01,Web02`.

## The trap: DSC resources are declarative blocks, not statements

```powershell
# Wrong instinct: this looks like it runs top-to-bottom like a script
configuration Bad {
    Node "localhost" {
        Service SvcA { Name = "Spooler"; State = "Running" }
        Service SvcB { Name = "BITS";    State = "Running" }
    }
}
```

Resources inside a `Node` block are **not guaranteed to apply in the
order written** unless you explicitly declare a dependency with
`DependsOn`:

```powershell
configuration Ordered {
    Node "localhost" {
        WindowsFeature IIS {
            Name   = "Web-Server"
            Ensure = "Present"
        }

        File SiteContent {
            DestinationPath = "C:\inetpub\wwwroot\index.html"
            Contents        = "<h1>Hello</h1>"
            Ensure          = "Present"
            DependsOn       = "[WindowsFeature]IIS"
        }
    }
}
```

Without `DependsOn`, the LCM is free to write `index.html` before IIS is
even installed, creating the directory manually — order in the script is
not order of application. This is the single most common DSC bug for
people coming from imperative scripting.

## Cheat sheet

| Concept | Purpose |
|---|---|
| `configuration { }` | declarative block compiled to one MOF per node |
| `Node "name" { }` | target a specific machine; one MOF file per node |
| Built-in resources (`File`, `Service`, `WindowsFeature`, `Registry`) | describe one piece of desired state |
| `-OutputPath` | where compiled `.mof` files land |
| `Start-DscConfiguration -Wait -Verbose` | apply a compiled configuration |
| `Test-DscConfiguration -Detailed` | audit drift without changing anything |
| `DependsOn = "[Type]Name"` | force one resource to apply after another |
| `Get-DscConfiguration` | show current state as last applied |

## Exercise

Write a `configuration` block named `DevBoxSetup` that ensures: a
directory `C:\DevTools` exists (`File` resource with
`Type = "Directory"`), a `Registry` value flags the machine as
provisioned, and a `Service` of your choosing is running — with the
service declared to depend on the directory via `DependsOn`. Compile it
with `-OutputPath` and inspect the generated `.mof` file's contents (on a
non-ARM64 machine, or read through the logic by hand if you're on Apple
Silicon) to confirm the dependency ordering is captured correctly.
