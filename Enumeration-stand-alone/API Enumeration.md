# API Enumeration and Abuse

> **PEN-200 Chapter 8 — §8.3.3.** In a black-box web test, API endpoints may expose application functionality that is not visible in the normal UI. Discover the paths, understand the normal request format, then test methods, authentication, and application logic.

## API mapping workflow

```text
Discover web/API port
      ↓
Find endpoint paths and versions
      ↓
Inspect responses / documentation if exposed
      ↓
Map parameters and JSON fields
      ↓
Identify authentication / tokens
      ↓
Test HTTP methods and authorization logic
      ↓
Replay interesting requests in Burp Repeater
```

Do not assume the web page exposes the full attack surface.

## Discover versioned API paths

Chapter 8 notes common patterns such as:

```text
/<api-name>/v1
/<api-name>/v2
```

A Gobuster pattern file can combine likely endpoint names with versions:

```text
{GOBUSTER}/v1
{GOBUSTER}/v2
```

```bash
gobuster dir -u http://<TARGET>:<PORT>/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -p <PATTERN_FILE>
```

After finding an endpoint, enumerate beneath it too:

```bash
gobuster dir -u http://<TARGET>:<PORT>/<API>/v1/<USER>/ \
  -w /usr/share/wordlists/dirb/small.txt
```

Also check for exposed documentation or UI paths discovered during content discovery.

## Inspect endpoints with curl

Show status, headers, and body:

```bash
curl -i http://<TARGET>:<PORT>/<API>/v1
```

JSON POST request:

```bash
curl -i \
  -H 'Content-Type: application/json' \
  -d '{"username":"<USER>","password":"<PASS>"}' \
  http://<TARGET>:<PORT>/<API>/v1/login
```

Use the exact property names and request format learned from the application responses or captured browser traffic.

## Treat status codes and response bodies as clues

### `405 Method Not Allowed`

A `405` is especially useful:

```text
405 Method Not Allowed
        ↓
Endpoint probably exists
        ↓
Current HTTP method is unsupported
        ↓
Test the method that matches the operation
```

PEN-200 demonstrates trying alternatives such as **POST** and **PUT** when GET is rejected. A response body can also reveal application behavior even when the HTTP status alone is ambiguous.

### Test method semantics

Chapter 8 specifically contrasts:

- **POST** — often used to submit/create data
- **PUT** / **PATCH** — often used when replacing/updating a value

Example method change:

```bash
curl -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Authorization: <SCHEME> <TOKEN>' \
  -d '{"password":"<NEW_VALUE>"}' \
  http://<TARGET>:<PORT>/<API>/v1/<USER>/password
```

Only send methods that make sense for the discovered endpoint and authorized lab scope.

## Authentication tokens

Chapter 8's example login returns a **JWT** authentication token. Record:

```text
token value
header / scheme used to send it
user identity
privilege level
expiration if relevant
```

Then replay authenticated requests with the same authorization format the application expects.

## Business-logic / authorization testing

Do not test only classic injection payloads. Compare what the application **should** allow with what the API actually accepts.

Questions to ask:

- Can a normal user supply a privilege-related property during registration?
- Does the server trust hidden/client-supplied JSON properties?
- Can one user access or modify another user's resource?
- Does a protected endpoint enforce authorization on every request?
- Does changing the HTTP method expose functionality the UI never offered?

The Chapter 8 lab demonstrates an API accepting an administrative property during registration and then allowing privileged API actions — a **logic/authorization flaw**, not an injection flaw.

## Move API testing into Burp

```text
Build/observe request
      ↓
Send through Burp proxy
      ↓
Repeater → modify method / JSON / auth header
      ↓
Target → Site map → keep discovered endpoints organized
```

```bash
curl -i http://<TARGET>/<API_PATH> --proxy 127.0.0.1:8080
```

## OSCP checklist

```text
[ ] Look for /api, versioned routes, /ui, docs, console-like paths
[ ] Enumerate beneath discovered API objects/users
[ ] Read JSON errors carefully — they reveal required properties and valid behavior
[ ] Distinguish 404 from 405 and inspect the response body
[ ] Recreate login/registration requests manually
[ ] Record auth token + required Authorization header format
[ ] Test GET / POST / PUT / PATCH only where evidence supports them
[ ] Test authorization/business logic, not just injection
[ ] Save interesting API requests in Burp Repeater / Site map
```

See also:

- [[Enumeration-stand-alone/Web enum|Web enumeration]]
- [[Enumeration-stand-alone/Burp Suite|Burp Suite]]
- [[Enumeration-stand-alone/XSS|Cross-Site Scripting]]

tag:#enumeration tag:#web tag:#api
