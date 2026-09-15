---
description: "Working with JSON/REST APIs — JSON is the universal data format for web APIs, config files, and inter-service communication — and PowerShell's object…"
---

# 07 · Working with JSON/REST APIs

JSON is the universal data format for web APIs, config files, and
inter-service communication — and PowerShell's object pipeline maps onto it
naturally. This module covers converting between PowerShell objects and
JSON, and calling REST APIs directly with `Invoke-RestMethod`, without
needing `curl` or a separate HTTP library.

## `ConvertTo-Json` and `ConvertFrom-Json`

```powershell
$config = @{
    Name    = "log-monitor"
    Version = "1.2.0"
    Enabled = $true
    Tags    = @("automation", "logs")
}

$json = $config | ConvertTo-Json
$json
```

```text
{
  "Version": "1.2.0",
  "Tags": [
    "automation",
    "logs"
  ],
  "Name": "log-monitor",
  "Enabled": true
}
```

```powershell
$obj = $json | ConvertFrom-Json
$obj.Name          # log-monitor
$obj.Tags[0]        # automation
$obj.GetType().Name # PSCustomObject
```

`ConvertFrom-Json` always produces `[pscustomobject]` instances (not
hashtables), so you access properties with dot notation just like any
other PowerShell object.

## The trap: `ConvertTo-Json`'s default depth is too shallow for nested data

```powershell
$nested = @{
    App = @{
        Name     = "monitor"
        Settings = @{ Retries = 3 }
    }
}

$nested | ConvertTo-Json -Depth 1
```

```text
WARNING: Resulting JSON is truncated as serialization has exceeded the set depth of 1.
{
  "App": {
    "Settings": "System.Collections.Hashtable",
    "Name": "monitor"
  }
}
```

`-Depth` defaults to 2 in most PowerShell versions, and nested hashtables
or custom objects deeper than that get silently flattened into their
`.ToString()` representation (`"System.Collections.Hashtable"`, not real
JSON) — with only a warning, not an error, so it's easy to miss in a
script's output. Always pass an explicit `-Depth` generous enough for your
data:

```powershell
$nested | ConvertTo-Json -Depth 5
```

```text
{
  "App": {
    "Settings": {
      "Retries": 3
    },
    "Name": "monitor"
  }
}
```

## `Invoke-RestMethod`: calling an API and getting objects back

```powershell
$todo = Invoke-RestMethod -Uri "https://jsonplaceholder.typicode.com/todos/1"

$todo.title        # delectus aut autem
$todo.completed     # False
$todo.GetType().Name # PSCustomObject
```

`Invoke-RestMethod` does the whole round trip: sends the HTTP request,
reads the response body, and — if the response is JSON — parses it into
PowerShell objects automatically. Compare this to `Invoke-WebRequest`,
which returns the raw HTTP response (status, headers, raw content string)
without parsing it for you; use `Invoke-RestMethod` when you know you're
talking to a JSON API, `Invoke-WebRequest` when you need the raw response
details.

## Sending data: POST with a JSON body

```powershell
$body = @{
    title  = "New Post"
    body   = "Hello from PowerShell"
    userId = 1
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://jsonplaceholder.typicode.com/posts" `
    -Method Post `
    -Body $body `
    -ContentType "application/json"

$response.id        # 101 (the API's fake "created" id)
$response.title     # New Post
```

Three things have to line up for a POST to work correctly: the `-Method`,
a `-Body` that's already a JSON **string** (not a raw hashtable —
`ConvertTo-Json` it first), and a `-ContentType` header telling the server
what format the body is in. Forgetting `-ContentType "application/json"`
is a common mistake — many APIs will silently misinterpret or reject the
body without it.

## Authentication headers

```powershell
$headers = @{
    Authorization = "Bearer $env:API_TOKEN"
    Accept        = "application/json"
}

Invoke-RestMethod -Uri "https://api.example.com/orders" -Headers $headers
```

Reading a token from an environment variable (`$env:API_TOKEN`) instead of
hardcoding it directly in the script keeps secrets out of source control —
a habit worth building early, even in throwaway scripts.

## Reading status codes and headers (PowerShell 7.4+)

```powershell
$result = Invoke-RestMethod -Uri "https://jsonplaceholder.typicode.com/todos/1" `
    -ResponseHeadersVariable respHeaders `
    -StatusCodeVariable statusCode

Write-Output "Status: $statusCode"
Write-Output "Content-Type: $($respHeaders['Content-Type'])"
```

```text
Status: 200
Content-Type: application/json; charset=utf-8
```

`-ResponseHeadersVariable` and `-StatusCodeVariable` populate named
variables as a side effect, since `Invoke-RestMethod`'s normal return value
is just the parsed body — this is how you inspect status/headers without
switching to `Invoke-WebRequest`.

## Handling API errors

```powershell
try {
    Invoke-RestMethod -Uri "https://jsonplaceholder.typicode.com/nonexistent-endpoint-xyz" -ErrorAction Stop
} catch {
    Write-Output "Caught: $($_.Exception.Message)"
    Write-Output "Status: $($_.Exception.Response.StatusCode.value__)"
}
```

```text
Caught: Response status code does not indicate success: 404 (Not Found).
Status: 404
```

`Invoke-RestMethod` raises a non-terminating error on non-2xx HTTP
responses (4xx/5xx), so — just like the rest of Level 2's error handling —
you need `-ErrorAction Stop` for `try/catch` to actually intercept it. The
`.Response.StatusCode` on the caught exception gives you the numeric status
code for branching logic (retry on 503, fail fast on 401, etc.).

## Cheat sheet

| Cmdlet/Parameter | Purpose |
|---|---|
| `ConvertTo-Json -Depth N` | serialize an object to a JSON string; always set `-Depth` explicitly for nested data |
| `ConvertFrom-Json` | parse a JSON string into `[pscustomobject]` |
| `Invoke-RestMethod` | call an API, get the parsed JSON body back directly |
| `Invoke-WebRequest` | call an API, get the raw response (status, headers, raw content) |
| `-Method Post/Put/Delete` | HTTP verb to use |
| `-Body` | request payload (JSON-encode it first) |
| `-ContentType "application/json"` | tells the server the body's format |
| `-Headers @{...}` | custom headers, e.g. `Authorization` |
| `-ResponseHeadersVariable` / `-StatusCodeVariable` | capture headers/status alongside the parsed body |
| `-ErrorAction Stop` | required for `try/catch` to catch a non-2xx response |

## How It Actually Works

`Invoke-RestMethod` is not `Invoke-WebRequest` with automatic parsing
bolted on for convenience — it inspects the response's `Content-Type`
header and dispatches to a format-specific deserializer (JSON via
`System.Text.Json`/`JavaScriptSerializer`, XML via `XmlDocument`, RSS/Atom
via feed parsing) and hands back **already-materialized PowerShell
objects**, whereas `Invoke-WebRequest` always gives you the raw
`HttpResponseMessage` wrapper with `.Content` as a byte/string payload you
must parse yourself. This is the real distinction, not "REST vs. web" as
the names suggest — you can call a JSON API with `Invoke-WebRequest` and
manually pipe `.Content` through `ConvertFrom-Json`, but `Invoke-
RestMethod` does that content-negotiation step for you.

`ConvertFrom-Json` parses text into either `PSCustomObject` (default) or,
with `-AsHashtable`, a `Hashtable`/`OrderedDictionary` tree — the default
`PSCustomObject` path matters because JSON objects have no fixed .NET
type, so each JSON object becomes a synthetic ETS-only object with
note properties for each JSON key, recursively, all the way down; property
*names* that aren't valid PowerShell identifiers still work via
`$obj.'weird-name'` because ETS property access doesn't require identifier
syntax, only dot-or-quote member access.

Authentication headers and body serialization both hinge on the same
content-negotiation logic: `-Body` combined with `-ContentType
'application/json'` sends the string as-is, but passing a hashtable to
`-Body` without JSON conversion sends it as `application/x-www-form-
urlencoded` key/value pairs instead — the cmdlet infers encoding from
what you hand it and the declared content type, not from any deep
inspection of your data's shape, which is the actual reason
`-Body (ConvertTo-Json $obj)` is written explicitly rather than relying on
implicit conversion.

Non-2xx responses throw a **terminating** `HttpResponseException`/
`WebException` from `Invoke-RestMethod`, unlike a plain socket call which
would just hand you the error body — this is why `try/catch` around API
calls is idiomatic here specifically, and why the error response body
(often containing the API's own error JSON) has to be extracted from
`$_.Exception.Response` rather than from a normal successful return value.

## 🔀 See this in another language

- [TypeScript — 08 · Working with JSON/APIs](https://sigilipelli.github.io/typescript-mastery-path/level-2/08-working-with-json-apis/)
- [Ruby — 06 · Working with JSON/APIs](https://sigilipelli.github.io/ruby-mastery-path/level-2/06-json-apis/)
- [PHP — 06 · Working with JSON/APIs](https://sigilipelli.github.io/php-mastery-path/level-2/06-json-apis/)

## Exercise

Write a function `Get-RandomJoke` that calls `https://official-joke-api.appspot.com/random_joke`,
returns a `[pscustomobject]` with just the `setup` and `punchline`
properties, and wraps the call in `try/catch` with `-ErrorAction Stop` so a
network failure prints a friendly `"Couldn't fetch a joke: ..."` message
instead of an unhandled exception. Call it 3 times in a loop and print each
result.
