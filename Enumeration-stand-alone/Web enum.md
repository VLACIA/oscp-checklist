# Web Enumeration

> **PEN-200 Chapter 8 additions.** Treat every HTTP(S) service as an application made of server technology, hidden content, requests/responses, headers, client-side code, APIs, and user-controlled inputs. The visible home page is only the starting point.

## Chapter 8 workflow

```text
Web port
  ↓
fingerprint server / stack
  ↓
discover hidden files and directories
  ↓
inspect source + DevTools + response headers
  ↓
proxy traffic through Burp
  ↓
map parameters / cookies / headers / APIs
  ↓
test application behavior and input handling
```

Detailed notes:

- [[Enumeration-stand-alone/Burp Suite|Burp Suite — Proxy, Repeater, Intruder, Site map]]
- [[Enumeration-stand-alone/API Enumeration|API Enumeration and Abuse]]
- [[Enumeration-stand-alone/XSS|Cross-Site Scripting (XSS)]]

 

**Immediately after Nmap finds HTTP/HTTPS** on 80, 443, 8000, 8080, 8443, etc.

```
# 1. Tech fingerprint
whatweb http://<TARGET_IP> -v
curl -I http://<TARGET_IP>
nikto -h http://<TARGET_IP> -output nikto.txt &

# 2. Directory brute force
gobuster dir -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
    -x php,html,txt,asp,aspx,jsp -t 50 -o gobuster.txt

feroxbuster -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,html,txt,aspx -t 100 --depth 3 -o ferox.txt

# 3. Always manually check:
curl http://<TARGET_IP>/robots.txt
curl http://<TARGET_IP>/.git/HEAD
curl http://<TARGET_IP>/.env
curl http://<TARGET_IP>/backup/
curl http://<TARGET_IP>/config.php
view-source:http://<TARGET_IP>/           # HTML comments!

# 4. VHost enumeration:
gobuster vhost -u http://<DOMAIN> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    --append-domain -t 50 -o vhosts.txt

# 5. CMS-specific:
wpscan --url http://<TARGET_IP> --enumerate u,p,t,cb,dbe --plugins-detection aggressive
joomscan -u http://<TARGET_IP>

# 6. Parameter fuzzing:
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -u "http://<TARGET_IP>/page.php?FUZZ=test" -fs 0 -mc 200,301,302,500
```

---

## PEN-200 Chapter 8 enumeration additions

### Nmap HTTP-specific enumeration

After confirming the web port/version, run the HTTP enumeration NSE script against the actual web port:

```bash
sudo nmap -p <PORT> --script=http-enum <TARGET_IP>
```

Treat discovered login, upload, database, JavaScript, image, or other directories as leads to investigate manually.

### Technology-stack clues

PEN-200 uses **Wappalyzer** to identify OS/web server/framework/JavaScript-library clues. Your existing `whatweb` step serves a similar role: record the stack and investigate interesting versions rather than treating the fingerprint as proof of a vulnerability.

### Browser Developer Tools

Use Firefox Developer Tools during manual enumeration:

- **Debugger** — inspect JavaScript/resources; use Pretty Print on minified code.
- **Inspector** — inspect HTML and quickly find hidden form fields/client-side controls.
- **Network** — refresh the page, inspect requests/responses and response headers.
- **Console** — test/debug JavaScript while analyzing client-side behavior.
- **Storage** — inspect cookies and flags when authentication/session behavior matters.

### Response headers

Look for technology disclosures such as:

```text
Server
X-Powered-By
X-AspNet-Version
x-amz-cf-id
```

Header names/values can reveal the server, framework, version, proxy/CDN, or other stack components. Also remember that request headers such as `User-Agent` can themselves become attacker-controlled application inputs.

### Sitemaps and robots

Keep the existing `robots.txt` check and also try sitemap discovery:

```bash
curl http://<TARGET_IP>/robots.txt
curl http://<TARGET_IP>/sitemap.xml
```

`robots.txt` exclusion entries and sitemap URLs can expose administrative, sensitive, or otherwise unlinked parts of the application.

### Manual application testing after discovery

Once the basic map exists, do not stop at directory enumeration:

```text
Interesting path / feature
      ↓
observe normal browser request
      ↓
[[Enumeration-stand-alone/Burp Suite|Burp Repeater]]
      ↓
change one parameter / header / method
      ↓
API or input behavior?
      ├─ [[Enumeration-stand-alone/API Enumeration|API Enumeration]]
      └─ [[Enumeration-stand-alone/XSS|XSS testing]]
```

tag:#enumeration tag:#web

---

## PEN-200 Chapter 9 — common web attack checks

After mapping the application and its parameters, use the Chapter 9 attack paths when the feature supports them:

```text
Filename / path parameter
  ├─ [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/Directory Traversal|Directory Traversal]]
  └─ [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/LFI-Local File Inclusion|LFI / RFI]]

Upload form
  └─ [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/File Upload Bypass|File Upload]]

Input that appears to invoke an OS utility
  └─ [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/Command Injection|Command Injection]]
```

### Fast Chapter 9 checklist

- Filename-valued parameters (`page=`, `file=`, `language=`): test `../`, over-traversal, Windows `..\`, and URL-encoded `%2e%2e/`.
- Linux file-read foothold: `/etc/passwd` → home directories → `.ssh/id_rsa` → SSH.
- Windows/IIS file-read targets: `hosts`, `web.config`, IIS log paths.
- LFI: test `php://filter` source disclosure, log poisoning, `data://`, and RFI when PHP configuration allows it.
- Uploads: identify where the file lands, whether it executes, alternate/case-changed extensions, and traversal in multipart `filename=`.
- Command-backed features: begin with accepted input, determine what command is actually executed, then test encoded separators and identify the underlying shell.

The Chapter 9 objective is not merely to prove a web bug; look for a chain from **web input → file/command control → credentials or code execution → foothold**.
