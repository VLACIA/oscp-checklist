# Windows Living-off-the-Land Enumeration

> **PEN-200 Chapter 6 — §6.3.** Use this when you are enumerating from a Windows workstation or compromised Windows host and Kali/Nmap or additional tools are unavailable. PEN-200 uses standard Windows utilities and PowerShell for this workflow.

## DNS with `nslookup`

```cmd
nslookup <HOST>.<DOMAIN>
nslookup -type=TXT <HOST>.<DOMAIN> <DNS_SERVER_IP>
```

Use confirmed hostnames and records as input for the next enumeration round.

## Test a TCP port with PowerShell

```powershell
Test-NetConnection -Port 445 <TARGET_IP>
Test-NetConnection -Port 25 <TARGET_IP>
```

`TcpTestSucceeded : True` indicates that the tested TCP port is reachable/open.

## Basic TCP port scan with built-in .NET classes

PEN-200 demonstrates scanning the first 1024 TCP ports with `TcpClient`:

```powershell
1..1024 | % {
    echo ((New-Object Net.Sockets.TcpClient).Connect("<TARGET_IP>", $_)) "TCP port $_ is open"
} 2>$null
```

This is a fallback when a normal port scanner is unavailable; extend or narrow the range according to the situation and scope.

## SMB shares with `net view`

```cmd
net view \\<HOST> /all
```

This can reveal normal and administrative shares such as:

```text
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
```

## SMTP from Windows

First confirm port 25:

```powershell
Test-NetConnection -Port 25 <TARGET_IP>
```

If the Telnet client is available:

```cmd
telnet <TARGET_IP> 25
```

Then test server-supported SMTP enumeration commands, for example:

```text
VRFY <USERNAME>
```

PEN-200 notes that enabling the Windows Telnet client may require administrative privileges.

## Workflow

```text
Windows foothold / client
        ↓
nslookup → discover names / DNS records
        ↓
Test-NetConnection / TcpClient → discover reachable TCP services
        ↓
net view → SMB resources
        ↓
Telnet / native clients → interact with discovered services
        ↓
Feed every new hostname, user, share, and service back into enumeration
```

**Key idea:** Living off the land means using tools already trusted/present on the host when installing your preferred tooling is not possible.
