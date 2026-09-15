# Chain A — Web Version CVE → Credentials → Privilege Escalation

**Attack path:** Web server/application version → known CVE → shell as `www-data` → credentials in configuration → SSH login → `sudo` abuse → root

## Brief explanation

1. **Fingerprint the web stack:** HTTP headers, error pages, source code, and application files may reveal the Apache, Nginx, CMS, framework, or plugin version.
2. **Exploit a matching CVE:** Confirm that the detected version and configuration are vulnerable, then use the flaw to gain remote code execution as the low-privileged web-service account, commonly `www-data`.
3. **Search application configuration:** Files such as `config.php`, `.env`, and database configuration files often contain plaintext usernames and passwords.
4. **Test credential reuse:** A database or application password may also be the operating-system user's SSH password.
5. **Escalate privileges:** Run `sudo -l`; an overly permissive `sudo` rule may provide a path to root, often using the documented technique for that binary in GTFOBins.

**Why the chain works:** An exposed software flaw provides the initial foothold, while poor secret storage, password reuse, and an unsafe `sudo` rule turn a restricted web shell into root access.
