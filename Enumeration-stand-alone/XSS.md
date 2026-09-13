# Cross-Site Scripting (XSS)

> **PEN-200 Chapter 8 — §8.4.** XSS occurs when attacker-controlled input is rendered as executable client-side code in another user's browser. The key question is not only "can JavaScript execute?" but **whose browser/session executes it and what that session can do**.

## Main XSS types

### Stored / Persistent XSS

```text
Attacker input
    ↓
stored in database/cache
    ↓
later rendered to another user
    ↓
JavaScript executes in that user's browser
```

Common locations include comments, reviews, or other content saved and displayed later.

### Reflected XSS

```text
crafted request / link
    ↓
input included in response
    ↓
victim opens/submits request
    ↓
payload executes in victim browser
```

Search results and error messages are common places to investigate.

### DOM-based XSS

The browser modifies the page's **DOM** using user-controlled data and injected JavaScript executes during client-side processing. DOM-based XSS can be stored or reflected.

## Identification workflow

Find every attacker-controlled input, not only visible form fields:

- query parameters
- POST/form fields
- JSON values
- hidden fields
- URL/path values
- HTTP headers such as `User-Agent`

Start by checking whether characters important to HTML/JavaScript survive filtering/encoding:

```text
< > ' " { } ;
```

Then inspect **where the value lands** in the response/DOM.

```text
Input reflected?
      ↓
Are special characters encoded / removed?
      ↓
What is the output context?
      ├─ HTML text / between tags
      ├─ attribute value
      └─ existing JavaScript
      ↓
Choose syntax that fits that context
```

PEN-200 emphasizes that the required characters change with context. For example, creating a new element requires angle brackets, while input already inside JavaScript may require quotes/semicolons instead.

## Basic confirmation

A simple proof used in Chapter 8 is:

```html
<script>alert(42)</script>
```

Use a harmless confirmation first. An alert proves script execution but does **not** represent the full impact.

## Test headers too

Chapter 8 demonstrates stored XSS through a user-controlled **User-Agent** value that is stored and later rendered without sanitization.

Burp workflow:

```text
Proxy → HTTP History
      ↓
Send request to Repeater
      ↓
Modify User-Agent (or another controllable value)
      ↓
Send
      ↓
Trigger the page that later renders the stored value
```

This is a reminder to fuzz inputs beyond the visible page UI.

## Why victim context matters

Injected JavaScript executes in the browser context of the user who views the vulnerable page. If an administrator renders stored attacker-controlled content, the script may be able to perform actions available to that administrative session.

Potential impact discussed in Chapter 8 includes:

- session theft when cookies are accessible
- redirection or page manipulation
- acting through a privileged user's authenticated session

## Cookie flags relevant to XSS

### `Secure`

Cookie is sent only over encrypted connections such as HTTPS.

### `HttpOnly`

Browser denies JavaScript access to the cookie.

```text
XSS exists
  + session cookie is HttpOnly
        ↓
direct document.cookie theft is blocked
        ↓
look for actions the victim session can perform instead
```

Chapter 8's WordPress example uses `HttpOnly` session cookies, so the attack changes direction rather than stopping at cookie theft.

## XSS + anti-CSRF nonce concept

A server-generated nonce can prevent a normal CSRF request because an outside attacker does not know the token. But JavaScript already executing **inside the victim's authenticated origin** can request a protected page, read the nonce from the response, then send a second same-origin request using that token.

Conceptual flow from Chapter 8:

```text
Stored XSS executes as admin
        ↓
GET protected admin page
        ↓
extract current nonce
        ↓
send authenticated admin action with nonce
        ↓
privileged application action succeeds
```

The chapter demonstrates this by using XSS executed by a WordPress administrator to create another administrative account. The important OSCP lesson is the **attack chain and security context**, not the alert box.

## OSCP checklist

```text
[ ] Locate every user-controllable input, including headers
[ ] Check whether input is reflected or stored
[ ] Try special characters and inspect encoding/filtering
[ ] Determine output context before choosing payload syntax
[ ] Confirm execution harmlessly
[ ] Identify which user role renders the payload
[ ] Inspect cookie flags (Secure / HttpOnly)
[ ] If cookie theft is blocked, test what same-origin actions the victim session can perform
[ ] Use Burp Repeater to mutate and reproduce the request
```

See also:

- [[Enumeration-stand-alone/Web enum|Web enumeration]]
- [[Enumeration-stand-alone/Burp Suite|Burp Suite]]
- [[Enumeration-stand-alone/API Enumeration|API Enumeration]]

tag:#web tag:#xss tag:#foothold
