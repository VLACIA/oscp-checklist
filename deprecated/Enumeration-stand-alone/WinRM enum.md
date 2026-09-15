# WinRM enumeration (5985/5986 TCP)

## Identify and validate

```bash
nmap -sV -p5985,5986 --script http-title,http-headers <TARGET_IP> -oN winrm.txt
curl -i http://<TARGET_IP>:5985/wsman
nxc winrm <TARGET_IP> -u <USER> -p '<PASSWORD>'
```

An HTTP `405` response from `/wsman` can still indicate a live WinRM endpoint. Port 5985 normally uses HTTP with message-level authentication; 5986 normally uses HTTPS.

## Obtain an authorized shell

```bash
evil-winrm -i <TARGET_IP> -u <USER> -p '<PASSWORD>'
# Hash authentication, when applicable:
evil-winrm -i <TARGET_IP> -u <USER> -H <NT_HASH>
```

- Record hostname/domain clues, TLS certificate names, authentication behavior, and whether the user is permitted remote management.
- Distinguish bad credentials from a valid user lacking WinRM authorization.
- Once connected, record identity, groups, privileges, language mode, and architecture before post-exploitation.
