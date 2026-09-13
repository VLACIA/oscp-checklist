# Passive information gathering / OSINT

> **PEN-200 Chapter 6 — §6.2.** Use passive information gathering to expand the attack surface before active probing. Information gathering is iterative: every new hostname, person, domain, technology, or credential can become input for another enumeration step.

## WHOIS

```bash
whois <DOMAIN>
whois <IP>
```

Record useful results such as:

- Registrar and registration dates
- Authoritative name servers
- Public registrant/admin/technical contacts
- Hosting organization, netblock, CIDR, and ownership information from IP WHOIS

The PEN-200 `whois ... -h <SERVER>` examples use an OffSec lab WHOIS server; for normal notes, the generic commands above are the portable form.

## Search-engine operators / Google hacking

Start broad, then narrow the query.

```text
site:<DOMAIN>
site:<DOMAIN> filetype:txt
site:<DOMAIN> ext:php
site:<DOMAIN> ext:xml
site:<DOMAIN> ext:py
site:<DOMAIN> -filetype:html
intitle:"index of" "parent directory"
```

Look for:

- `robots.txt` and paths not obvious from normal browsing
- Directory listings
- Documents, configuration files, backups, and non-HTML resources
- File extensions that reveal languages/frameworks

**Reference:** Google Hacking Database (GHDB) contains many combined operator examples.

## Public source-code repositories

Search GitHub, GitHub Gist, GitLab, and SourceForge for the target organization, users, project names, domains, and interesting filenames.

Possible findings:

- Programming languages and frameworks
- Usernames
- Password hashes
- Credentials, API tokens, and cloud access keys
- Internal hostnames and paths

For larger repositories, PEN-200 mentions **Gitrob** and **Gitleaks** to automate secret discovery. These tools can miss findings, so manual inspection still matters.

## Third-party internet intelligence

### Netcraft

Use it to collect:

- Subdomains / hostnames
- Site history and registration context
- Web technologies
- Related hosts / netblocks

### Shodan

Useful for a passive snapshot of internet-connected assets:

- IP addresses
- Open ports
- Service banners and versions
- Technologies
- Published vulnerabilities associated with identified services

PEN-200 treats Shodan as useful background material, not a requirement for the module labs.

### Security Headers / SSL Labs

Use third-party checks to identify security-posture clues:

- Missing HTTP defensive headers such as CSP or `X-Frame-Options`
- Legacy TLS versions / weak cipher configuration
- SSL/TLS-related issues

Missing hardening controls are not automatically exploitable vulnerabilities, but they can indicate weaker server-hardening practices.

## What to add to the target map

```text
Domains / subdomains
IP ranges / hosting providers
Nameservers / mail servers
Employees / usernames / email formats
Technologies / service versions
Interesting indexed files and directories
Credentials / hashes / tokens
Potential cloud assets
New hosts to actively enumerate
```

Then feed every useful discovery back into DNS, port scanning, service enumeration, credential testing, and later foothold/lateral-movement work.
