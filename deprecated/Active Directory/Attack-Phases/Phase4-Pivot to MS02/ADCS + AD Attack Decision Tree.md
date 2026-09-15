**Run certipy find immediately with provided creds**
        │
        ▼
┌──────────────────────────────┐
│ Vulnerable template found?   │
└──────────────┬───────────────┘
               │
       ┌───────┴──────────────────────┐
       │                              │
      Yes                             No
       │                              │
       ▼                              ▼
     ESC1                        No ADCS vuln
       │                              │
       ▼                              ▼
certipy req -upn Administrator   Continue normal AD path
       │
       ▼
certipy auth → NTLM hash
       │
       ▼
**PTH as Administrator**
       │
       ▼
**Domain Admin ✓**


ESC8 (HTTP endpoint)?
       │
       ▼
certipy relay + PetitPotam coerce
       │
       ▼
DC cert
       │
       ▼
DCSync
       │
       ▼
ALL hashes
       │
       ▼
Golden Ticket