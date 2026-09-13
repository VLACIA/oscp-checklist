# Enumeration Plan

**Enumeration = finding doors before you try to kick them in.** The goal is to build a complete picture of the target: hosts, ports, software, versions, names, users, credentials, and every new clue that can expand the attack surface.

**PEN-200 Chapter 6 key idea:** information gathering is **cyclical**, not a one-time phase. New information discovered during reconnaissance, foothold, or lateral movement should trigger another enumeration round.

## 0. Passive discovery first (when useful and in scope)

[[Enumeration-stand-alone/Passive OSINT|Passive OSINT]]

Collect domains, subdomains, IP ranges, nameservers, people/usernames, public code, technologies, secrets, and third-party service intelligence before direct probing.

## 1. Active network discovery

[[Enumeration-stand-alone/nmap|Nmap workflow]] → [[Enumeration-stand-alone/Port Scanning Theory|TCP / UDP Port Scanning Theory]]

**The #1 student mistake:** running one quick `nmap` scan, seeing port 80, and tunnel-visioning on the website for hours while another service provides the real foothold. Scan broadly enough to understand the attack surface before committing to one path.

## 2. Enumerate every discovered service

For every service, ask:

1. **What exactly is it?** Version, product, OS/host role, hostname/domain.
2. **What is exposed without credentials?** Shares, files, users, banners, records, APIs.
3. **What does this discovery suggest next?** CVE research, default/anonymous credentials, password reuse, virtual hosts, new hosts, or protocol-specific enumeration.

## 3. Vulnerability scanning / candidate generation

After identifying reliable service/version information, use [[Enumeration-stand-alone/Vulnerability Scanning|Vulnerability Scanning]] to generate or validate vulnerability candidates. Automated results are **not proof**: account for false positives, false negatives, backported patches, scanner visibility, and intrusive checks before choosing an exploit.

## 4. Windows-only fallback

If you are enumerating from a Windows host without Kali/Nmap, use [[Enumeration-stand-alone/Windows LOTL Enumeration|Windows Living-off-the-Land Enumeration]].

## 5. Feed discoveries back into the loop

```text
new hostname → DNS + Nmap + service enumeration
new username → SMTP / SMB / authentication checks
new credential → validate only on relevant discovered services
new IP range → host discovery + reverse DNS
new software/version → targeted vulnerability research
new local-only service → local enumeration / pivoting decision
```

**Golden rule:** do not treat enumeration as complete simply because the first scan finished.

tag:#enumeration
