# Chain B — Anonymous SMB → Credentials → Remote Access → Privilege Escalation

**Attack path:** SMB null/guest session → readable share → sensitive document or configuration → valid credentials → SSH/WinRM → local privilege escalation

## Brief explanation

1. **Enumerate SMB without credentials:** A server may permit a null session or guest access and disclose share names, users, or files.
2. **Inspect readable shares:** Configuration files, scripts, backups, PDFs, and spreadsheets may contain passwords or other secrets.
3. **Validate the credentials:** Test discovered credentials only against relevant exposed services, such as SSH on Linux or WinRM on Windows.
4. **Escalate locally:** On Linux, look for unsafe SUID programs, `sudo` rules, or other misconfigurations. On Windows, privileges such as `SeImpersonatePrivilege` may enable elevation to `SYSTEM` when a suitable technique applies.

**Why the chain works:** Anonymous file access becomes an authenticated foothold because sensitive credentials were stored in a broadly readable share; a separate host misconfiguration then enables elevation.
