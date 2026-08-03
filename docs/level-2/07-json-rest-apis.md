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

## Exercise

Write a function `Get-RandomJoke` that calls `https://official-joke-api.appspot.com/random_joke`,
returns a `[pscustomobject]` with just the `setup` and `punchline`
properties, and wraps the call in `try/catch` with `-ErrorAction Stop` so a
network failure prints a friendly `"Couldn't fetch a joke: ..."` message
instead of an unhandled exception. Call it 3 times in a loop and print each
result.
