# SMB / NetBIOS enumeration (139/445 TCP)

SMB (TCP 445) and NetBIOS (commonly TCP 139 plus UDP services) are separate protocols, although they are often enumerated together.

```bash
# Find SMB / NetBIOS across a range
nmap -v -p139,445 -oG smb.txt <CIDR>

# NetBIOS names can reveal host roles
sudo nbtscan -r <CIDR>

# Unauthenticated SMB
smbclient -L //<TARGET_IP> -N
smbmap -H <TARGET_IP> -u '' -p ''
nxc smb <TARGET_IP> -u '' -p '' --shares
nmap --script smb-vuln* -p445 <TARGET_IP>

# SMB OS/domain discovery (legacy SMBv1-dependent in PEN-200 example)
nmap -v -p139,445 --script smb-os-discovery <TARGET_IP>

# Authenticated
nxc smb <TARGET_IP> -u user -p pass --shares --users --groups --pass-pol
smbmap -H <TARGET_IP> -u user -p pass -R
smbclient //<TARGET_IP>/share -U 'user%pass'

# Download all files recursively
smbclient //<TARGET_IP>/SHARE -U 'user%pass' -c 'recurse ON; prompt OFF; mget *'
nxc smb <TARGET_IP> -u user -p pass -M spider_plus
```

Useful SMB/NSE output can include computer name, NetBIOS name, domain, forest, FQDN, and system time. Treat OS guesses as clues; PEN-200 demonstrates an inaccurate Windows version result.

From a Windows foothold/client:

```cmd
net view \\<HOST> /all
```

See [[Enumeration-stand-alone/Windows LOTL Enumeration|Windows LOTL Enumeration]].
