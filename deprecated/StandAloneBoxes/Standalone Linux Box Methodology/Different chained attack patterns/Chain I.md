# Chain I — Disclosed Username Used as Password

**Attack path:** username disclosure → predictable password → authenticated login → local privilege escalation

## Brief explanation

1. **Collect valid usernames:** SMTP user enumeration, SNMP data, SMB shares, web pages, error messages, and document metadata can reveal account names.
2. **Test predictable credentials carefully:** Poor password policies may allow values derived from the account or organization, such as `username`, `username123`, or a company-themed password.
3. **Authenticate to an exposed service:** A valid pair may provide access through SSH, a web portal, SMB, or another service available to that account.
4. **Enumerate privilege escalation:** Treat the login as the initial foothold and look for platform-appropriate local misconfigurations or vulnerable software.

**Why the chain works:** Username disclosure is usually low impact by itself, but it becomes serious when users choose predictable passwords and the host lacks controls such as strong password policy, rate limiting, or multi-factor authentication.

> [!note]
> Keep attempts minimal and account-lockout-aware. This pattern is a targeted check for weak credentials, not unrestricted brute force.
