What these tools do

**BloodHound** collects every user, group, and permission in the domain and draws it as a graph — then you click "Shortest Path to Domain Admins" and it literally shows you the route. Run it _first_, every time. **nxc** (NetExec) quickly lists shares, users, and the password policy. **ldapsearch** reads the AD directory directly — and admins infamously leave passwords sitting in user "description" fields, one of the most common OSCP quick wins.

```
# BloodHound — FIRST THING TO RUN:
bloodhound-python -u stephanie -p 'Password123' \
    -d corp.local -ns 192.168.x.100 -c All --zip
# Import ZIP → run queries:
#   "Find Shortest Paths to Domain Admins"
#   "Find All Kerberoastable Users"
#   "Find AS-REP Roastable Users"
#   "Computers with Unconstrained Delegation"
#   "Find Principals with DCSync Rights"

# nxc domain recon:
nxc smb 192.168.x.100 -u stephanie -p 'Password123' \
    --shares --users --groups --pass-pol

# Passwords in description field (VERY COMMON in OSCP):
ldapsearch -x -H ldap://192.168.x.100 \
    -D "stephanie@corp.local" -w 'Password123' \
    -b "DC=corp,DC=local" \
    "(&(objectClass=user)(description=*))" sAMAccountName description
```