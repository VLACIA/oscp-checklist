# FTP enumeration (21/TCP)

> Use only against systems you are authorized to test. Record the banner, server version, authentication result, permissions, and interesting files.

## Identify and connect

```bash
nmap -sV -sC -p21 --script "ftp-*" <TARGET_IP> -oN ftp.txt
ftp <TARGET_IP>
# Common anonymous login: anonymous / anonymous
```

## Check

- Test anonymous access and whether uploads, downloads, directory creation, or deletion are allowed.
- Recursively inspect exposed files for configs, backups, usernames, credentials, web roots, and transfer mode issues.
- If upload is allowed, determine where the file is written and whether another exposed service serves or executes it. Do not assume upload means code execution.
- Note FTPS/TLS support and search the exact product/version only after confirming the banner.

Useful client commands: `ls -la`, `binary`, `passive`, `get`, `mget`, `put`.
