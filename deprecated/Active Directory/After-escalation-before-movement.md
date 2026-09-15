# After Escalation — Before Movement

**Local admin / SYSTEM / root achieved** → do **not** immediately jump to the next host. Chapter 24 repeatedly emphasizes re-enumerating with the new privilege because previously inaccessible credentials, files, sessions, and network information may now be visible.

```text
privileged shell
    ↓
re-enumerate locally
    ↓
loot credentials / sessions / configs / history
    ↓
map routes + interfaces + internal hosts
    ↓
update creds + BloodHound / host notes
    ↓
choose the movement method that matches the loot
```

## 1. Re-run local enumeration as the privileged user

Linux:

```bash
./linpeas.sh
```

Windows:

```powershell
.\winPEAS.exe
```

Do not assume the first low-privilege scan saw everything.

## 2. Credential material

### Windows

Prioritize:

```text
LSASS logon sessions
LSA secrets
cached domain credentials / mscache
DPAPI material
Kerberos tickets
saved RDP / PowerShell / application credentials
```

If a **privileged domain user has an active session on this host**, the host becomes a high-value credential target.

Chapter 24's pattern:

```text
SYSTEM on server
  + Domain Admin session/cache on server
  → Mimikatz / LSASS
  → plaintext password or NTLM
  → lateral movement to DC
```

Mimikatz commands used in the chapter:

```text
privilege::debug
sekurlsa::logonpasswords
```

### Linux

Hunt:

```text
keytabs / ccache
SSH keys + passphrases
shell history
service credentials
LDAP bind credentials
web/database configs
```

## 3. Configuration + developer artifact hunting

Search the files that root/admin can read but your original user could not:

```text
web.config / wp-config.php / .env
PowerShell history / bash_history
backup files
scripts / scheduled jobs
Git repositories and deleted commits
sshpass / rsync / scp automation
```

For local Git history:

→ [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/git Exposed → Credential Chain|Git → Credential Chain]]

## 4. Who is logged on / recently active?

Windows examples:

```cmd
quser
net session
```

Correlate local findings with BloodHound session data:

→ [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]]

High-value logic:

```text
you control HOST as admin/SYSTEM
+
privileged user has session on HOST
=
dump credentials here before moving elsewhere
```

## 5. Network interfaces / routes / pivot value

Check whether the newly privileged host exposes new reachability. A machine may be **dual-homed** or have routes/interfaces that were not relevant from the previous foothold.

Record:

```text
IP / subnet / gateway
DNS
additional interfaces
internal routes
known/cached hosts
```

If new internal reachability appears, update your pivot plan:

→ [[Active Directory/Pivoting-tunneling/Intro|Pivoting overview]] · [[Active Directory/Pivoting-tunneling/Chisel|Chisel]]

## 6. Update the working set

For every new secret:

```text
username : password/hash/ticket
source host
source file/process/session
where validated
privileges confirmed
```

For every new host:

```text
IP : hostname : role : reachable services : signing / admin notes
```

Then mark newly controlled identities/computers as **Owned** in BloodHound and re-run path analysis.

## 7. Move with what you actually looted

```text
plaintext password → direct login / spray only where appropriate
NTLM hash          → Pass-the-Hash
Kerberos ticket    → Pass-the-Ticket
new route          → SOCKS / Chisel / Ligolo
admin session      → LSASS / token / credential theft
```

Related:

- [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/Lateral Movement — Pass-the-Hash|Pass-the-Hash]]
- [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/pass-the-ticket|Pass-the-Ticket]]
- [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/AD Lateral Movement — WMI, WinRM, PsExec & DCOM|AD Lateral Movement]]
- [[Basics/Password Attacks/NTLM Capture & Relay|NTLM Capture & Relay]]
- [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]]

## Minimal checklist

```text
[ ] re-run linPEAS / winPEAS as privileged user
[ ] dump / inspect credential material
[ ] check active privileged sessions
[ ] grep configs + histories + developer artifacts
[ ] inspect Git history / deleted scripts
[ ] map interfaces / routes / new internal hosts
[ ] record every credential + source
[ ] validate new identities
[ ] mark Owned + re-run BloodHound
[ ] then move using password / hash / ticket / route
```
