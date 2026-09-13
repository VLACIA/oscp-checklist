# SSH enumeration (22/TCP)

## Fingerprint

```bash
nmap -sV -p22 --script ssh2-enum-algos,ssh-hostkey,ssh-auth-methods <TARGET_IP> -oN ssh.txt
ssh -vvv <USER>@<TARGET_IP>
```

## Check

- Record the OpenSSH/product version, host keys, supported authentication methods, and whether usernames or banners disclose useful context.
- Reuse only credentials and private keys found during authorized enumeration; fix key permissions with `chmod 600 id_rsa`.
- For an encrypted private key, identify the format and crack offline when permitted (`ssh2john` then John).
- Try the correct username, domain form, key, and non-default port before assuming valid material failed.
- Treat old algorithms as a compatibility clue, not proof of a vulnerability; enable a legacy algorithm only for the specific connection if required.

Do not brute-force SSH unless the engagement permits it and the lockout/noise risk is understood.
