# Windows Privilege Escalation Plan

## Mental model

**Privilege escalation = you already have a Windows foothold as a lower-privileged user and are looking for a path to an administrative user or `NT AUTHORITY\SYSTEM`.** Chapter 16 emphasizes combining **manual situational awareness**, sensitive-information discovery, Windows-service abuse, scheduled tasks, privileges/tokens, and carefully selected exploits.

Do not treat privilege escalation as a one-pass checklist. **Every time you obtain a new user or security context, repeat enumeration** because the new account may have different file access, group memberships, service permissions, or credentials available.

## Chapter 16 workflow

```text
Windows foothold
    ↓
[[Windows Enumeration and Sensitive Information|1. Enumerate Windows + hunt sensitive information]]
    ↓
Re-evaluate after every new credential / user
    ↓
┌──────────────────────────────────────────────────────────────┐
│ Check all applicable privilege-escalation paths              │
├──────────────────────────────────────────────────────────────┤
│ [[Token Priviledge|Token privileges / UAC / integrity]]      │
│ [[Windows Service Abuse|Service binary / DLL / unquoted]]    │
│ [[Scheduled Tasks and Exploits|Scheduled tasks / exploits]]  │
│ Stored credentials / config files / PowerShell artifacts     │
└──────────────────────────────────────────────────────────────┘
    ↓
Administrator / SYSTEM
    ↓
[[Dump Credentials (Windows Post-Exploitation)|Dump credentials / post-exploitation]]
```

## First commands after a foothold

```powershell
whoami
hostname
whoami /groups
whoami /priv
systeminfo
ipconfig /all
route print
netstat -ano
```

Then continue with the full manual workflow in [[Windows Enumeration and Sensitive Information]].

## Priority checks

**1. Situational awareness + credentials** — identify users/groups, OS/build/architecture, routes/connections, installed applications, running processes, configuration files, documents, PowerShell history/transcripts, and password-manager artifacts.

**2. Token privileges** — `whoami /priv`. `SeImpersonatePrivilege` is especially valuable; Chapter 16 demonstrates PrintSpoofer. Also note `SeBackupPrivilege`, `SeAssignPrimaryToken`, `SeLoadDriver`, and `SeDebugPrivilege` as potentially dangerous privileges.

**3. Windows services** — enumerate service paths and permissions. Check for writable service binaries, missing/writable DLLs, and unquoted paths with writable candidate directories.

**4. Scheduled tasks** — identify the principal, trigger, and action. A task running as SYSTEM/admin becomes interesting when the executed program/script is writable and the trigger is usable.

**5. Application/kernel exploits** — use only after matching the exact application/OS context. Kernel exploits can crash the target, so prefer lower-risk misconfigurations and credential paths first when possible.

## Automated enumeration

Use **winPEAS** to save time, but **verify findings manually**. Chapter 16 shows that automated tools can miss useful files/history/transcripts or even report incorrect system details. If winPEAS is blocked, PEN-200 also mentions **Seatbelt** and **JAWS**, or fall back to manual enumeration.

```powershell
.\winPEAS.exe
```

For service-focused checks, see [[Windows Service Abuse#PowerUp — useful, but verify manually]].
