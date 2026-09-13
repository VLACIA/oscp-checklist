# SMTP enumeration (25/465/587 TCP)

## Fingerprint and inspect

```bash
nmap -sV -p25,465,587 --script smtp-commands,smtp-enum-users,smtp-open-relay <TARGET_IP> -oN smtp.txt
nc -nv <TARGET_IP> 25
# EHLO test.local
```

For implicit TLS use `openssl s_client -connect <TARGET_IP>:465`; for STARTTLS use `openssl s_client -starttls smtp -connect <TARGET_IP>:587`.

## Check

- Record the banner, hostname/domain, supported commands, authentication mechanisms, and TLS certificate names.
- Where supported, carefully test `VRFY`, `EXPN`, and `RCPT TO` responses for user enumeration.
- Use `smtp-user-enum -M VRFY|RCPT|EXPN -U users.txt -t <TARGET_IP>` only when authorized and rate-limited.
- Verify suspected users across other services and build a username list.
- Test open relay only with a controlled sender and recipient you own; do not deliver mail to third parties.
