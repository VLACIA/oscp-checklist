# Burp Suite — manual web testing workflow

> **PEN-200 Chapter 8 — §8.2.4 and §8.3.3.** Burp Suite is the main manual web proxy used in the chapter to inspect, modify, replay, and organize HTTP requests.

## Start and proxy setup

```bash
burpsuite
```

Default Burp listener used in PEN-200:

```text
127.0.0.1:8080
```

Configure Firefox to use the listener as its manual proxy. When simply browsing/mapping the application, leave **Intercept off** so requests flow normally. Turn **Intercept on** when you need to stop and edit a request before it reaches the server.

```text
Firefox / client
      ↓
Burp Proxy : 127.0.0.1:8080
      ↓
Target web application
```

## Proxy → HTTP History

Use **Proxy → HTTP History** as the request/response log while browsing the application.

For each interesting request, record:

- HTTP method and path
- query/body parameters
- cookies / authentication material
- request headers
- status code
- response headers
- response body / length

A browser-side restriction is not a security boundary. Burp lets you alter request values, parameter names, form values, and headers before the server processes them.

## Repeater — primary manual testing tool

```text
Proxy → HTTP History
        ↓
Right-click interesting request
        ↓
Send to Repeater
        ↓
Modify one thing at a time
        ↓
Send → compare response
```

Use Repeater to test:

- parameter values
- headers such as `User-Agent`
- cookies / auth headers
- GET vs POST vs PUT behavior
- JSON request bodies
- hidden fields or values the browser UI does not expose

The response pane shows the raw server response, including headers and unrendered content.

## Intruder — automate a selected input

Chapter 8 demonstrates Intruder by selecting one login password field and iterating a small wordlist.

Workflow:

```text
Send request to Intruder
        ↓
Positions → Clear automatic positions
        ↓
Select only the value to vary → Add
        ↓
Payloads → load values / wordlist
        ↓
Start Attack
        ↓
Compare status / length / response differences
```

**OSCP note:** first understand one normal request in Repeater. Automate only the specific value you actually want to vary.

## Target → Site map

After testing endpoints through Burp:

```text
Target → Site map
```

Use the Site map to organize discovered application/API paths and resend saved requests to **Repeater** or **Intruder** for additional testing.

## Curl through Burp

PEN-200 also sends command-line requests through Burp so the exact request can be inspected:

```bash
curl -i http://<TARGET>/<PATH> --proxy 127.0.0.1:8080
```

This is useful when you build an API request with `curl` but want Burp to capture and modify it.

## Fast decision rule

```text
Interesting browser/API request?
        ↓
Capture in HTTP History
        ↓
Send to Repeater
        ↓
Understand normal response
        ↓
Change one parameter / header / method
        ↓
Interesting repeatable behavior?
        ├─ yes → continue manually / map in Site map
        └─ repetitive input → consider Intruder
```

See also:

- [[Enumeration-stand-alone/Web enum|Web enumeration]]
- [[Enumeration-stand-alone/API Enumeration|API Enumeration]]
- [[Enumeration-stand-alone/XSS|Cross-Site Scripting]]

tag:#enumeration tag:#web tag:#burp
