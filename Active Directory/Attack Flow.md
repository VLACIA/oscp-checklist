**PROVIDED CREDS (stephanie:Password123)**
        │
        ▼
**Phase 1: RAPID AD ENUMERATION**
BloodHound + nxc + LDAP + PowerView
        │
        ▼
**Phase 2: QUICK WINS (all simultaneously)**
├── AS-REP Roasting
├── Kerberoasting
├── Passwords in descriptions (VERY COMMON)
├── SMB share hunting
└── GPP/SYSVOL passwords
        │
        ▼
**Phase 3: FOOTHOLD on MS01**
(local.txt = 10 pts)
        │
        ▼
**Phase 4: PIVOT to MS02**
(local.txt = 10 pts)
Setup Ligolo-ng tunnel → lateral move
        │
        ▼
**Phase 5: ESCALATE to Domain Admin**
ACL abuse / DCSync / Delegation / GPO
        │
        ▼
**Phase 6: OWN DC**
(proof.txt = 20 pts)
PSExec/WMIexec as DA → dump NTDS → Golden Ticket