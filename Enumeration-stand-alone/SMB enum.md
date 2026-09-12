```
# Unauthenticated:
smbclient -L //<TARGET_IP> -N
smbmap -H <TARGET_IP> -u '' -p ''
nxc smb <TARGET_IP> -u '' -p '' --shares
nmap --script smb-vuln* -p 445 <TARGET_IP>

# Authenticated:
nxc smb <TARGET_IP> -u user -p pass --shares --users --groups --pass-pol
smbmap -H <TARGET_IP> -u user -p pass -R
smbclient //<TARGET_IP>/share -U 'user%pass'

# Download all files recursively:
smbclient //<TARGET_IP>/SHARE -U 'user%pass' -c 'recurse ON; prompt OFF; mget *'
nxc smb <TARGET_IP> -u user -p pass -M spider_plus
```