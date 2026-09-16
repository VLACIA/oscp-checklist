---
title: "PEN-200 Chapter 6 - Information Gathering"
aliases:
  - "PEN200 Ch6 Information Gathering"
  - "OSCP Information Gathering"
tags:
  - oscp
  - pen200
  - reconnaissance
  - enumeration
  - nmap
  - dns
  - smb
  - smtp
  - snmp
source: "PEN-200 - Penetration Testing with Kali Linux, Chapter 6"
chapter: 6
status: study-note
---

# PEN-200 Chapter 6 - Information Gathering

> [!important] OSCP mindset
> Information gathering is **not a one-time opening phase**. It is iterative: every new hostname, port, service, username, technology, or network range should trigger another round of enumeration. The chapter's central habit is: **discover -> record -> enumerate deeper -> use the new information to discover more**.

## Coverage checklist

- [x] 6.1 The Penetration Testing Lifecycle
- [x] 6.2 Passive Information Gathering
- [x] 6.2.1 Whois Enumeration
- [x] 6.2.2 Google Hacking
- [x] 6.2.3 Netcraft
- [x] 6.2.4 Open-Source Code
- [x] 6.2.5 Shodan
- [x] 6.2.6 Security Headers and SSL/TLS
- [x] 6.3 Active Information Gathering
- [x] 6.3.1 DNS Enumeration
- [x] 6.3.2 TCP/UDP Port Scanning Theory
- [x] 6.3.3 Port Scanning with Nmap
- [x] 6.3.4 SMB Enumeration
- [x] 6.3.5 SMTP Enumeration
- [x] 6.3.6 SNMP Enumeration
- [x] 6.4 Wrapping Up

## Chapter in one picture

```text
Scope
  |
  v
Passive Recon / OSINT
  |-- WHOIS
  |-- Search engines / Google dorks
  |-- Netcraft / Shodan ---> shodan is passive because we look into the data shodan already collected 
  |-- Public source-code repositories
  |-- Security headers / TLS posture (TLS certificate → discover hostnames/vhosts. HTTP/security headers → learn about the application/server. Security headers learn about security configuration)
  |
  v
Active Enumeration
  |-- DNS -> hosts / subdomains / ranges
  |-- Port scanning -> TCP + UDP services
  |-- Nmap -> versions / OS / scripts
  |-- SMB -> names / shares / domain clues
  |-- SMTP -> usernames
  |-- SNMP -> users / processes / software / local ports
  |
  v
New targets and clues
  |
  +----> repeat enumeration cycle
  |
  v
Initial Access -> Privilege Escalation -> Credentials -> Pivoting -> AD -> Proof
```

---

# 6.1 The Penetration Testing Lifecycle


Treat your notes as a growing attack-surface database. For each host, track at minimum:

```text
IP / hostname
OS guess
Open TCP ports
Open UDP ports
Service + version
Web virtual hosts / URLs
Usernames
Credentials / hashes
Interesting files / shares
DNS names / domains
Potential vulnerabilities / attack paths
What has already been tested
```

---

# 6.2 Passive Information Gathering

Passive information gathering (OSINT) collects publicly available information before aggressively interacting with the target.

---

# 6.2.1 Whois Enumeration

## Tool: `whois`

WHOIS provides registration and ownership information for domain names and IP ranges. Useful fields can include:

- registrar
- registrant / organization
- contact information
- authoritative name servers
- IP netrange / CIDR
- hosting or allocation organization

### Domain lookup

```bash
whois megacorpone.com -h 192.168.50.251 
```

**Arguments**

- `megacorpone.com` - domain being queried.
- `-h 192.168.50.251` -arbitrary, send the query to this specific WHOIS server (`-h` = host). Fixed value!

**Why/when to use it**

Use WHOIS early in reconnaissance when you have a domain name. It may reveal nameservers, personnel, registration organization, and other infrastructure clues that can become inputs for DNS enumeration or OSINT.

### IP WHOIS / reverse ownership lookup

```bash
whois 38.100.193.70 -h 192.168.50.251
```

This asks who owns or hosts the IP range containing the supplied address. The output can reveal `NetRange`, `CIDR`, organization, and location information.

**OSCP connection:** On exam machines you are usually given target IPs directly, so WHOIS may be less important than in a real external engagement. The underlying habit still matters: determine **who owns an address/range and what related infrastructure exists** before expanding a scan.

---

# 6.2.2 Google Hacking

Google hacking (Google dorking) uses search operators to find indexed information that normal broad searches may hide among irrelevant results.

## Important operators

### `site:`

```text
site:megacorpone.com
```

Restricts results to a domain and its indexed subdomains/pages.

**Use it for:** understanding web presence, finding subdomains/pages, and creating a starting inventory.

### `filetype:` / `ext:`

```text
site:megacorpone.com filetype:txt
ext:php
ext:xml
ext:py
```

- `filetype:txt` - only TXT results.
- `ext:php`, `ext:xml`, `ext:py` - look for specific extensions/technologies.

The chapter's TXT example finds `robots.txt`, which can disclose paths that are not obvious from normal navigation.

### Exclusion with `-`

```text
site:megacorpone.com -filetype:html
```

The leading `-` removes matching results. This example removes ordinary HTML pages so that potentially more interesting non-HTML content stands out.

### `intitle:` and quoted strings

```text
intitle:"index of" "parent directory"
```

Looks for directory listings. Exposed directory indexes can reveal backups, configuration files, source files, logs, or other sensitive material.

## Tools/resources

### Google Hacking Database (GHDB)

A catalog of pre-built search queries/dorks grouped around interesting exposures and misconfigurations.

**When to use:** when you know the target technology or information type you want but need ideas for useful search patterns.

### DorkSearch

A portal with pre-built dorks and a query builder.

**When to use:** to quickly compose or experiment with operator combinations.

> [!tip] OSCP use
> Google dorking is more relevant to realistic external assessments than to many isolated exam boxes, but the thinking style is highly relevant: filter noisy data until only high-value information remains.

---

# 6.2.3 Netcraft

## Tool/service: Netcraft (website)

Netcraft is a third-party web service that can provide information such as:

- discovered hostnames/subdomains
- hosting/netblock information
- site history
- operating system indications
- web server technologies
- application/server technologies

Example target search in the chapter uses Netcraft's DNS search for `megacorpone.com` and then opens a **site report** for discovered hosts.

**Why/when to use it**

Use Netcraft during passive recon to learn about web infrastructure without directly scanning the target yourself. The resulting subdomains and technology fingerprints become inputs for later active enumeration.

**OSCP habit:** every newly found hostname goes into your notes and should later be resolved and scanned if it is in scope.

---

# 6.2.4 Open-Source Code

Public source-code platforms can reveal:

- programming languages
- frameworks and dependencies
- configuration conventions
- usernames
- internal hostnames
- API keys/tokens
- passwords/hashes
- cloud credentials

## Platforms mentioned

- GitHub
- GitHub Gist
- GitLab
- SourceForge

### GitHub search operator

```text
filename:users
```

Searches for files containing `users` in the filename. The chapter finds `xampp.users`, which contains a username and password hash.

**When to use:** after identifying an organization's GitHub account or code repositories. Search for filenames and keywords such as `users`, `password`, `secret`, `token`, `config`, `.env`, cloud-provider names, internal domains, and application names.

## Tools: Gitrob and Gitleaks

These automate secret discovery in repositories. They commonly use regular expressions and/or entropy detection to identify strings that resemble keys, tokens, and passwords.

The chapter's Gitleaks figure shows syntax in this form (the repository in the screenshot is redacted):

```bash
./gitleaks-linux-amd64 -v -r=https://github.com/<repo>
```

**Arguments shown**

- `-v` - verbose output.
- `-r=<repo URL>` - repository to inspect (syntax shown by the chapter's Gitleaks version).

**Why/when to use**

- Use manually on small repositories where context matters.
- Use Gitrob/Gitleaks on large repositories or many repositories to quickly find candidates.
- **Always manually inspect important repos too.** Automated secret scanners can miss data that does not match their patterns.

> [!important] Credential connection
> A public-repo credential can skip several attack steps. A password/hash/API key discovered during recon may become **Initial Access** or feed later **Credentials** reuse testing.

---

# 6.2.5 Shodan

## Tool/service: Shodan

Shodan searches internet-connected devices rather than mainly indexing web-page content. It can expose:

- IP addresses
- open ports
- service banners
- product/version information
- operating-system hints
- known/published vulnerability associations

### Example search filter

```text
hostname:megacorpone.com
```

This restricts Shodan results to systems associated with the hostname/domain.

**Why/when to use it**

Use Shodan before active scanning to get a passive snapshot of the public attack surface. If it identifies SSH, HTTP, VPN, database, or other services and their versions, you can prioritize later direct enumeration.

> [!note]
> Shodan data can be stale. Treat it as a clue, not proof. Verify important findings with active enumeration when permitted.

---

# 6.2.6 Security Headers and SSL/TLS

This section uses third-party services that perform checks against the target on your behalf. The chapter treats them as part of passive-style reconnaissance because the tester is not directly initiating the probes.

## Security Headers

The Security Headers service analyzes HTTP response headers and highlights missing protections such as:

- `Content-Security-Policy`
- `X-Frame-Options`

Missing headers are not automatically an exploitable vulnerability, but they can indicate weak hardening practices and suggest that other configuration mistakes may exist.

## Qualys SSL Labs - SSL Server Test

Analyzes SSL/TLS configuration, including:

- supported protocol versions
- cipher suites
- certificate/configuration quality
- some known TLS-related weaknesses (for example, POODLE or Heartbleed-related conditions)

The chapter's example identifies legacy TLS 1.0/1.1 support and weak/legacy cryptographic configuration.

**Why/when to use**

Use these findings to assess the target's security maturity and to decide where deeper web/TLS enumeration is worth your time.

---

# 6.3 Active Information Gathering

Active information gathering directly probes target systems. The chapter focuses on:

- DNS
- TCP/UDP port scanning
- Nmap
- SMB/NetBIOS
- SMTP
- SNMP
- Windows "Living off the Land" enumeration

## Living off the Land (LOLBAS/LOLBins idea)

Sometimes you do not have Kali or cannot install tools on a compromised Windows machine. In that case, use built-in or trusted Windows utilities to enumerate from the new network position.

Examples in this chapter:

- `nslookup`
- `Test-NetConnection`
- PowerShell/.NET `TcpClient`
- `net view`
- `dism`
- `telnet`

**Attack-chain relevance:** after initial access, these tools let you repeat reconnaissance from inside the network, which is the bridge to **pivoting and Active Directory discovery**.

---

# 6.3.1 DNS Enumeration

DNS often exposes the logical structure of an environment.

## DNS record types to know

| Record | Meaning | Pentest value |
|---|---|---|
| `NS` | Authoritative name servers | Identifies DNS infrastructure |
| `A` | Hostname -> IPv4 | Finds target IPs |
| `AAAA` | Hostname -> IPv6 | Finds IPv6 targets |
| `MX` | Mail servers | Finds mail infrastructure |
| `PTR` | IP -> hostname | Reverse discovery of hosts |
| `CNAME` | Alias to another name | Reveals related hosts/services |
| `TXT` | Arbitrary text | Verification data, email/security configuration, occasional clues |

## Tool: `host`

### Basic A-record lookup

```bash
host www.megacorpone.com
```

No explicit record type means `host` performs the normal address lookup (A record in this example).

### Query MX records

```bash
host -t mx megacorpone.com
```

- `-t mx` - request MX records only.

**Why:** identify mail servers and hostnames for later SMTP enumeration.

### Query TXT records

```bash
host -t txt megacorpone.com
```

- `-t txt` - request TXT records.

**Why:** TXT records can reveal verification strings, mail-security records, domain clues, and other metadata.

### Compare valid vs. invalid hostnames

```bash
host www.megacorpone.com
host idontexist.megacorpone.com
```

An invalid public DNS name returns `NXDOMAIN`. This response difference enables subdomain brute forcing.

## DNS brute forcing with Bash + `host`

### View a wordlist

```bash
cat list.txt
```

Example values in the chapter: `www`, `ftp`, `mail`, `owa`, `proxy`, `router`.

### Forward-lookup brute force

```bash
for ip in $(cat list.txt); do host $ip.megacorpone.com; done
```

**Pieces**

- `$(cat list.txt)` - command substitution; feeds every word to the loop.
- `$ip.megacorpone.com` - constructs each candidate FQDN.
- `host` - tests whether DNS resolves the candidate.

**Why:** discover valid subdomains/hostnames.

### Install larger wordlists

```bash
sudo apt install seclists
```

Installs SecLists, commonly available under `/usr/share/seclists`.

**Why:** use realistic, larger DNS/subdomain wordlists instead of a tiny hand-written list.

## Reverse DNS brute force

```bash
for ip in $(seq 200 254); do host 51.222.169.$ip; done | grep -v "not found"
```

**Pieces**

- `seq 200 254` - generates integers 200 through 254.
- `host 51.222.169.$ip` - reverse-resolves each IP.
- `|` - pipe output to the next command.
- `grep -v "not found"` - `-v` inverts the match, removing failed lookups.

**Why:** once several discovered hosts cluster in the same subnet, PTR records may reveal many additional hostnames such as `admin`, `intranet`, `vpn`, `snmp`, `syslog`, etc.

## Tool: DNSRecon

### Standard enumeration

```bash
dnsrecon -d megacorpone.com -t std
```

- `-d megacorpone.com` - target domain.
- `-t std` - standard/general DNS enumeration.

**Why:** automate collection of common record types and basic DNS information.

### Brute-force subdomains

```bash
cat list.txt
dnsrecon -d megacorpone.com -D ~/list.txt -t brt
```

- `-d` - target domain.
- `-D ~/list.txt` - dictionary/wordlist file.
- `-t brt` - brute-force enumeration type.

**Why:** automate the manual `host` loop and find additional hostnames.

## Tool: DNSenum

```bash
dnsenum megacorpone.com
```

With just the domain, DNSenum performs automated DNS enumeration, including host discovery/brute forcing and reverse lookup activity depending on the environment/tool behavior.

**Why:** quickly expand the known host and network inventory. The chapter demonstrates that automation can discover many hosts and related class-C ranges, after which you should repeat enumeration against the new targets.

## Windows tool: `nslookup`

### Simple hostname resolution

```powershell
nslookup mail.megacorptwo.com
```

Queries the system's configured DNS server.

### Query a specific record from a specific DNS server

```powershell
nslookup -type=TXT info.megacorptwo.com 192.168.50.151
```

- `-type=TXT` - request a TXT record.
- `info.megacorptwo.com` - hostname/record to query.
- `192.168.50.151` - DNS server to ask explicitly.

**Why:** very useful after compromise when you are on Windows and need internal DNS information without installing tools.

### OSCP workflow after DNS discovery

```text
new hostname -> resolve IP -> add to /etc/hosts if needed -> scan ports -> enumerate every exposed service
new subnet/range -> host discovery -> targeted service scans -> deeper full scans on interesting hosts
```

---

# 6.3.2 TCP/UDP Port Scanning Theory

Port scanning discovers which network services are reachable and therefore which attack surfaces exist.

## TCP CONNECT scan concept

TCP normally uses a three-way handshake:

```text
Client -> SYN -> Server
Client <- SYN/ACK <- Server
Client -> ACK -> Server
```

If the handshake completes, the port is open. A refused/reset connection usually indicates closed.

## Tool: Netcat (`nc`) - TCP scan

```bash
nc -nvv -w 1 -z 192.168.50.152 3388-3390
```

**Arguments**

- `-n` - do not perform DNS/name resolution; use numeric addresses.
- `-v` / `-vv` - verbose / more verbose output.
- `-w 1` - 1-second timeout.
- `-z` - zero-I/O scanning mode; connect without sending application data.
- `192.168.50.152` - target.
- `3388-3390` - port range.

**Why/when:** Netcat is not a full scanner, but it is commonly available and is useful for quick checks or when a dedicated scanner is unavailable.

## UDP scanning concept

UDP is connectionless; there is no three-way handshake. A scanner often infers:

- **ICMP Port Unreachable received** -> UDP port is closed.
- **No response** -> port may be open **or filtered**.
- **Application response** -> port is open.

This ambiguity makes UDP scanning less reliable.

## Netcat UDP scan

```bash
nc -nv -u -z -w 1 192.168.50.149 120-123
```

**Arguments**

- `-u` - UDP mode.
- `-n` - numeric output/no DNS lookup.
- `-v` - verbose.
- `-z` - scan/zero-I/O mode.
- `-w 1` - one-second timeout.
- `120-123` - UDP port range.

## Tool: Wireshark

The chapter uses Wireshark captures to show what is actually happening on the wire:

- TCP: SYN -> SYN/ACK or RST/ACK.
- UDP: empty UDP probes; closed ports often answer with ICMP Port Unreachable.

**Why it matters:** understanding packet behavior helps you interpret scanner output instead of blindly trusting `open`, `closed`, `filtered`, or `open|filtered` states.

> [!warning] Common UDP mistake
> Do not skip UDP. Services such as SNMP, NTP, DNS, and others may expose critical information or attack paths. Also remember that firewalls dropping ICMP can create false positives.

---

# 6.3.3 Port Scanning with Nmap

## Tool: Nmap

Nmap is the chapter's primary port scanner and service-enumeration framework. Some scan types use raw sockets and therefore require `sudo`/root privileges.

## Measuring scan traffic with `iptables`

The chapter deliberately measures how much traffic different Nmap scans create. iptables -L shows traffic rules.

### Insert rules to count target traffic

```bash
sudo iptables -I INPUT 1 -s 192.168.50.149 -j ACCEPT
sudo iptables -I OUTPUT 1 -d 192.168.50.149 -j ACCEPT
```

**Arguments**

- `-I INPUT 1` - insert a rule as rule 1 in the INPUT chain.
- `-I OUTPUT 1` - insert a rule as rule 1 in the OUTPUT chain.
- `-s <IP>` - source address match.
- `-d <IP>` - destination address match.
- `-j ACCEPT` - ACCEPT target/action.

These rules also provide counters that can be inspected after scanning.

### Reset counters

```bash
sudo iptables -Z
```

- `-Z` - zero packet/byte counters.

### Show counters

```bash
sudo iptables -vn -L
```

- `-v` - verbose output/counters.
- `-n` - numeric output (avoid name resolution).
- `-L` - list chains/rules.

**Why:** compare the network footprint of different scanning strategies.

## Default Nmap scan

```bash
nmap 192.168.50.149
```

Scans Nmap's default set of the 1000 most common TCP ports. With raw-socket privileges, Nmap normally uses SYN scanning; without them, it may fall back to a TCP connect scan.

**When:** quick first pass against a single host.

## Full TCP port scan

```bash
nmap -p 1-65535 192.168.50.149
```

- `-p 1-65535` - scan all TCP ports.

**Why:** services may run on high/nonstandard ports. The chapter shows that a full scan discovers ports missed by the default top-1000 scan, but produces far more traffic and takes longer.

> [!tip] OSCP habit
> A common workflow is a fast/default scan first, then a **full TCP scan in the background** while you begin enumerating the already-discovered services.

## Other scanners mentioned: MASSCAN and RustScan

Both are modern high-speed scanners. The chapter notes that they can generate substantial concurrent traffic, while Nmap includes more conservative rate behavior and extensive enumeration features.

**When to use:** when speed is necessary and you understand the network/traffic implications. On OSCP, Nmap remains the most important tool because discovery and detailed service enumeration are tightly integrated.

## SYN / stealth scan

```bash
sudo nmap -sS 192.168.50.149
```

- `-sS` - TCP SYN scan.
- `sudo` - needed for raw packets.

Nmap sends SYN and interprets the response without completing the normal handshake. It is faster and uses fewer packets than a full connect scan.

**Important:** "stealth" is historical terminology. Modern firewalls/IDS can detect SYN scans.

## TCP connect scan

```bash
nmap -sT 192.168.50.149
```

- `-sT` - TCP connect scan using the operating system's socket API.

**When:** when raw-socket privileges are unavailable or when scanning through some proxy types. It completes full TCP connections and is generally slower/noisier at the application level.

## UDP scan

```bash
sudo nmap -sU 192.168.50.149
```

- `-sU` - UDP scan.

Nmap may send generic empty probes or protocol-aware probes for well-known services such as SNMP.

## Combined TCP SYN + UDP scan

```bash
sudo nmap -sU -sS 192.168.50.149
```

- `-sU` - UDP.
- `-sS` - TCP SYN.

**Why:** build a more complete service picture than TCP-only scanning.

## Network sweep / host discovery

```bash
nmap -sn 192.168.50.1-253
```

- `-sn` - host discovery only; do not perform the normal port scan.

The chapter notes that Nmap host discovery can use several probe types rather than relying only on ICMP echo.

**When:** identify live hosts before spending time on detailed scans.

## Greppable output

```bash
nmap -v -sn 192.168.50.1-253 -oG ping-sweep.txt
grep Up ping-sweep.txt | cut -d " " -f 2
```

**Nmap arguments**

- `-v` - verbose.
- `-sn` - host discovery.
- `-oG ping-sweep.txt` - write greppable output to a file.

**Pipeline arguments**

- `grep Up` - keep lines containing `Up`.
- `cut -d " " -f 2` - split on spaces and print field 2 (the IP in this output format).

**Why:** create a clean list of live hosts for follow-up scanning.

## Sweep for a specific service/port

```bash
nmap -p 80 192.168.50.1-253 -oG web-sweep.txt
grep open web-sweep.txt | cut -d" " -f2
```

- `-p 80` - scan only TCP port 80.
- `-oG` - greppable output.

**Why:** quickly find systems running a service of interest. A service sweep can be more useful than generic ping discovery when ICMP is filtered.

## Top ports + aggressive enumeration

```bash
nmap -sT -A --top-ports=20 192.168.50.1-253 -oG top-port-sweep.txt
```

- `-sT` - TCP connect scan.
- `-A` - enables aggressive enumeration features (including service/version detection, OS-related detection, default script scanning, and traceroute where applicable).
- `--top-ports=20` - scan the 20 most common ports according to Nmap's service-frequency data.
- `-oG` - save greppable output.

**When:** a broad but relatively constrained sweep when you want more information than simple port state.

## Nmap service frequency database

```bash
cat /usr/share/nmap/nmap-services
```

Shows Nmap's service/port/protocol/frequency database, which influences "top ports" selection.

## OS fingerprinting

```bash
sudo nmap -O 192.168.50.14 --osscan-guess
```

- `-O` - OS fingerprinting.
- `--osscan-guess` - show guesses even when Nmap is not confident enough for an exact match.

**When:** you need an OS family/version hypothesis to guide later enumeration or exploit research.

> [!warning]
> OS fingerprinting is a **guess**. Firewalls, proxies, NAT, and packet rewriting can reduce accuracy.

## Service/version and aggressive scan

```bash
nmap -sT -A 192.168.50.14
```

- `-sT` - TCP connect scan.
- `-A` - aggressive discovery/enumeration.

The example identifies a FileZilla FTP service and Windows-related services.

For plain service/version detection, the chapter specifically notes:

```bash
nmap -sV <target>
```

- `-sV` - probe open ports to determine service/product/version.

**OSCP importance:** this is one of the most useful Nmap options because exploit research is usually driven by the **actual service and version**, not merely the port number.

## Nmap Scripting Engine (NSE)

NSE scripts live under:

```text
/usr/share/nmap/scripts
```

They automate discovery, enumeration, brute force, and some vulnerability checks.

### HTTP headers NSE script

```bash
nmap --script http-headers 192.168.50.6
```

- `--script http-headers` - run the `http-headers` NSE script.

**Why:** retrieve HTTP response-header information such as server software and content metadata.

### Script help

```bash
nmap --script-help http-headers
```

- `--script-help <script>` - display the script's purpose, category, documentation link, and usage information.

**Why:** use this whenever you find an unfamiliar NSE script. If internet access is unavailable, inspect the script file under `/usr/share/nmap/scripts`.

## Windows built-in port checking: `Test-NetConnection`

```powershell
Test-NetConnection -Port 445 192.168.50.151
```

- `-Port 445` - test TCP port 445.
- `192.168.50.151` - target.
- `TcpTestSucceeded : True` - confirms the TCP connection succeeded.

**Why:** check a specific port from a compromised/limited Windows machine without Nmap.

## PowerShell/.NET TCP port scan

```powershell
1..1024 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"} 2>$null
```

**Pieces**

- `1..1024` - generate ports 1 through 1024.
- `| % { ... }` - pipe into PowerShell's `ForEach-Object` alias `%`.
- `$_` - current port number.
- `New-Object Net.Sockets.TcpClient` - instantiate a .NET TCP client.
- `.Connect("192.168.50.151", $_)` - attempt the TCP connection.
- `2>$null` - discard error-stream output from failed connections.

**Why:** lightweight internal port scan when living off the land. The chapter chooses `TcpClient` because repeatedly using `Test-NetConnection` would generate additional unnecessary traffic.

---

# 6.3.4 SMB Enumeration

SMB commonly uses TCP/445. Legacy NetBIOS-over-TCP environments may also expose TCP/139 and UDP NetBIOS services.

SMB enumeration can reveal:

- host/computer names
- domains/forests
- shares
- user/group/domain information (depending on access and configuration)
- OS clues
- Active Directory-related context

## Scan SMB/NetBIOS ports across a network

```bash
nmap -v -p 139,445 -oG smb.txt 192.168.50.1-254
cat smb.txt
```

- `-v` - verbose.
- `-p 139,445` - scan NetBIOS session service and SMB.
- `-oG smb.txt` - save greppable output.
- `192.168.50.1-254` - target range.

**Why:** identify hosts worth deeper SMB enumeration.

## Tool: `nbtscan`

```bash
sudo nbtscan -r 192.168.50.0/24
```

- `-r` - use UDP source port 137 (NetBIOS name service), as described in the chapter.
- `192.168.50.0/24` - target subnet.

**Why:** discover NetBIOS names. Hostnames such as `DC`, `FILESERVER`, `BACKUP`, `SAMBAWEB`, etc. can immediately reveal a machine's likely role.

## Find SMB-related NSE scripts

```bash
ls -1 /usr/share/nmap/scripts/smb*
```

- `-1` - one entry per line.
- `smb*` - shell wildcard matching SMB scripts.

Examples listed in the chapter include:

```text
smb2-capabilities.nse
smb2-security-mode.nse
smb2-time.nse
smb2-vuln-uptime.nse
smb-brute.nse
smb-double-pulsar-backdoor.nse
smb-enum-domains.nse
smb-enum-groups.nse
smb-enum-processes.nse
smb-enum-sessions.nse
smb-enum-shares.nse
smb-enum-users.nse
smb-os-discovery.nse
```

## SMB OS/domain discovery with NSE

```bash
nmap -v -p 139,445 --script smb-os-discovery 192.168.50.152
```

- `-v` - verbose.
- `-p 139,445` - focus on SMB/NetBIOS.
- `--script smb-os-discovery` - run the SMB OS-discovery NSE script.

The script may reveal:

- OS guess
- computer name
- NetBIOS name
- domain name
- forest name
- FQDN
- system time

**Important limitation from the chapter:** this particular discovery approach depends on SMBv1 and may be inaccurate; do not blindly trust the reported OS.

**AD connection:** domain and forest names are extremely valuable. Once you learn the AD domain, record it immediately for later Kerberos/LDAP/SMB/AD enumeration.

## Windows tool: `net view`

```cmd
net view \\dc01 /all
```

- `\\dc01` - remote computer.
- `/all` - include administrative/hidden shares as well.

The chapter's example shows shares such as `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, and `SYSVOL`.

**Why:** enumerate remote SMB shares from Windows using a built-in command.

> [!tip] OSCP SMB habit
> Finding TCP/445 should trigger a dedicated SMB checklist: hostname/domain, SMB version/security mode, anonymous/null access if applicable, shares, readable files, users/groups, and credentials you can test later. Chapter 6 introduces the discovery side; later PEN-200 material goes deeper.

---

# 6.3.5 SMTP Enumeration

SMTP can leak valid usernames through commands such as:

- `VRFY <user>` - ask the server whether a mailbox/user exists.
- `EXPN <list>` - ask the server to expand a mailing list.

Not every SMTP server supports these commands, but when enabled they can provide usernames for later password attacks or credential reuse testing.

## Connect to SMTP with Netcat

```bash
nc -nv 192.168.50.8 25
```

- `-n` - numeric output/no DNS lookup.
- `-v` - verbose.
- `192.168.50.8` - mail server.
- `25` - SMTP port.

Then interact manually:

```text
VRFY root
VRFY idontexist
```

Different SMTP response codes/messages reveal whether the user likely exists.

## Python SMTP VRFY script from the chapter

```python
#!/usr/bin/python
import socket
import sys

if len(sys.argv) != 3:
    print("Usage: vrfy.py <username> <target_ip>")
    sys.exit(0)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
ip = sys.argv[2]
connect = s.connect((ip,25))
banner = s.recv(1024)
print(banner)
user = (sys.argv[1]).encode()
s.send(b'VRFY ' + user + b'\r\n')
result = s.recv(1024)
print(result)
s.close()
```

### Run it

```bash
python3 smtp.py root 192.168.50.8
python3 smtp.py johndoe 192.168.50.8
```

**Arguments**

- first argument - username to test.
- second argument - SMTP server IP.

**Why:** automate the same `VRFY` interaction performed manually with Netcat.

## Windows: check SMTP port

```powershell
Test-NetConnection -Port 25 192.168.50.8
```

Confirms that TCP/25 is reachable, but does not provide full interactive SMTP enumeration.

## Enable Windows Telnet client

```powershell
dism /online /Enable-Feature /FeatureName:TelnetClient
```

- `/online` - service the currently running Windows installation.
- `/Enable-Feature` - enable an optional Windows feature.
- `/FeatureName:TelnetClient` - feature to enable.

**Requirement:** administrative privileges.

## Interact with SMTP using Telnet on Windows

```cmd
telnet 192.168.50.8 25
```

Then:

```text
VRFY goofy
VRFY root
```

**Why:** perform service-level enumeration from a Windows foothold when Kali is unavailable.

---

# 6.3.6 SNMP Enumeration

SNMP is a particularly valuable enumeration target because it exists to expose management information. Older versions (especially SNMPv1/v2c) commonly use community strings instead of strong authentication and do not encrypt traffic.

Common default community strings include:

```text
public
private
manager
```

The chapter's major lesson is that SNMP can expose **far more than network status**: usernames, processes, installed applications, system information, and locally listening ports.

## SNMP Management Information Base (MIB)

The MIB is a tree of Object Identifiers (OIDs). Querying specific branches retrieves specific categories of data.

### Windows OIDs highlighted in the chapter

| OID | Information |
|---|---|
| `1.3.6.1.2.1.25.1.6.0` | System processes |
| `1.3.6.1.2.1.25.4.2.1.2` | Running programs |
| `1.3.6.1.2.1.25.4.2.1.4` | Process paths |
| `1.3.6.1.2.1.25.2.3.1.4` | Storage units |
| `1.3.6.1.2.1.25.6.3.1.2` | Installed software names |
| `1.3.6.1.4.1.77.1.2.25` | User accounts |
| `1.3.6.1.2.1.6.13.1.3` | TCP local/listening ports |

## Find open SNMP services with Nmap

```bash
sudo nmap -sU --open -p 161 192.168.50.1-254 -oG open-snmp.txt
```

- `-sU` - UDP scan.
- `--open` - show only open ports.
- `-p 161` - scan SNMP's standard UDP port 161.
- `192.168.50.1-254` - network range.
- `-oG open-snmp.txt` - greppable output file.

**Why:** find SNMP targets before testing community strings.

## Tool: `onesixtyone`

First create a community-string list:

```bash
echo public > community
echo private >> community
echo manager >> community
```

- `>` - create/overwrite file.
- `>>` - append to file.

Build an IP list:

```bash
for ip in $(seq 1 254); do echo 192.168.50.$ip; done > ips
```

Then test combinations:

```bash
onesixtyone -c community -i ips
```

- `-c community` - file containing community strings.
- `-i ips` - file containing target IPs.

**Why:** efficiently identify hosts that accept a known/default SNMP community string.

## Tool: `snmpwalk`

### Enumerate the full accessible MIB tree

```bash
snmpwalk -c public -v1 -t 10 192.168.50.151
```

- `-c public` - community string.
- `-v1` - SNMP version 1.
- `-t 10` - 10-second timeout.
- `192.168.50.151` - target.

**Why:** broad SNMP dump when you have a valid community string. It may reveal system description, hostname, contact email, interfaces, and many other objects.

### Enumerate Windows user accounts

```bash
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.4.1.77.1.2.25
```

The final argument is the user-account OID.

**Why:** discover usernames that can feed SMB, RDP, WinRM, Kerberos, password guessing, or credential-reuse tests later.

### Enumerate running processes

```bash
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.4.2.1.2
```

**Why:** identify services/applications and security software. A process name may reveal a vulnerable product or defensive control.

### Enumerate installed software

```bash
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.25.6.3.1.2
```

**Why:** correlate software names with running processes and versions to identify potential attack paths.

### Enumerate locally listening TCP ports

```bash
snmpwalk -c public -v1 192.168.50.151 1.3.6.1.2.1.6.13.1.3
```

**Why:** this can reveal ports/services that are listening only locally and were therefore invisible in your external Nmap scan. That information becomes extremely important after you gain code execution, set up port forwarding, or begin pivoting.

> [!important] Pivoting connection
> SNMP may tell you that a service exists even when you cannot reach it directly. Record it. Later, after obtaining a foothold, you may be able to tunnel/forward to that local-only service.

---

# 6.4 Wrapping Up

The chapter closes with three practical lessons:

1. **Enumeration is iterative.** Passive and active techniques feed each other.
2. **There is no universal best tool.** Many tools overlap; become comfortable with several and understand their differences.
3. **Understand what tools actually do.** Measure traffic, study packets, and question scanner guesses rather than accepting output blindly.

For OSCP, the practical version is:

```text
Enumerate broadly -> identify promising services -> enumerate each service deeply ->
record new users/hosts/ports/credentials -> loop back -> exploit only after you understand the surface.
```

---

# Attack chain connection

The requested OSCP chain is:

```text
Recon -> Enumeration -> Initial Access -> Privilege Escalation -> Credentials -> Pivoting -> AD -> Proof
```

Chapter 6 lives mainly in **Recon + Enumeration**, but its output is what drives every stage after that.

| Attack-chain stage | How Chapter 6 contributes |
|---|---|
| **Recon** | WHOIS, Google dorks, Netcraft, source-code search, Shodan, security-header/TLS services build the initial external picture. |
| **Enumeration** | DNS, Nmap, SMB, SMTP, SNMP, and Windows built-ins turn IPs/hostnames into specific services, users, shares, versions, domains, and internal clues. |
| **Initial Access** | Service/version data tells you where to investigate vulnerabilities or weak configurations. Public repo secrets or SMTP/SNMP usernames may directly enable authentication attacks. |
| **Privilege Escalation** | Chapter 6 is not primarily a privilege-escalation chapter, but enumeration habits carry forward: identify OS, software, processes, local ports, and configuration clues before choosing a privesc path. |
| **Credentials** | Git repositories may expose hashes/tokens; SMTP and SNMP may expose usernames; SMB can reveal domain context; these become inputs for later credential attacks. |
| **Pivoting** | After foothold, repeat enumeration from the compromised host using `nslookup`, `Test-NetConnection`, `net view`, PowerShell `TcpClient`, etc. SNMP can reveal local-only services that may be reached through tunnels. |
| **AD** | DNS/SMB clues such as domain name, forest name, domain-controller-like ports, `NETLOGON`, and `SYSVOL` are signals that the target is part of AD. These findings become the entry point to later AD enumeration. |
| **Proof** | Good enumeration notes tell you exactly which host/service path produced compromise. Keep evidence, commands, usernames, ports, and discoveries organized so you can reproduce and document the path to proof. |

## What to carry from Chapter 6 into every OSCP machine

```text
1. Scan TCP.
2. Do not forget UDP.
3. Identify exact services/versions.
4. Enumerate every open service individually.
5. Resolve and record hostnames/domains.
6. Add every new username to a user list.
7. Add every credential/hash/token to a credential list.
8. Treat SMB/DNS/SNMP as information gold mines.
9. Re-scan/re-enumerate after obtaining a foothold or new network position.
10. Never trust one scanner result blindly; verify important findings.
```

---

# Command and tool quick reference by purpose

## Passive recon

| Goal | Tool/query |
|---|---|
| Domain registration | `whois <domain>` |
| IP ownership/netrange | `whois <ip>` |
| Search target domain | `site:target.tld` |
| Find file types | `site:target.tld filetype:txt` / `ext:php` |
| Remove noise | `site:target.tld -filetype:html` |
| Directory listings | `intitle:"index of" "parent directory"` |
| Search repository filenames | `filename:users` |
| Search internet-facing devices | `hostname:target.tld` in Shodan |
| Website technology/history | Netcraft |
| Secret discovery | Gitrob / Gitleaks |
| HTTP header posture | Security Headers |
| TLS posture | Qualys SSL Labs |

## Active discovery/enumeration

| Goal | Command/tool |
|---|---|
| DNS A lookup | `host host.target.tld` |
| MX lookup | `host -t mx target.tld` |
| TXT lookup | `host -t txt target.tld` |
| DNS brute force | `dnsrecon -d target.tld -D wordlist -t brt` |
| Automated DNS enum | `dnsenum target.tld` |
| Windows DNS | `nslookup <name>` |
| Quick TCP check | `nc -nvv -w 1 -z <IP> <ports>` |
| Quick UDP check | `nc -nv -u -z -w 1 <IP> <ports>` |
| Default Nmap | `nmap <IP>` |
| Full TCP | `nmap -p 1-65535 <IP>` |
| SYN scan | `sudo nmap -sS <IP>` |
| Connect scan | `nmap -sT <IP>` |
| UDP | `sudo nmap -sU <IP>` |
| Service versions | `nmap -sV <IP>` |
| OS guess | `sudo nmap -O <IP> --osscan-guess` |
| Host discovery | `nmap -sn <range>` |
| Windows port test | `Test-NetConnection -Port <port> <IP>` |
| SMB scan | `nmap -p 139,445 <IP/range>` |
| NetBIOS names | `sudo nbtscan -r <subnet>` |
| SMB shares (Windows) | `net view \\host /all` |
| SMTP interaction | `nc -nv <IP> 25` + `VRFY <user>` |
| Find SNMP | `sudo nmap -sU --open -p 161 <range>` |
| Community brute force | `onesixtyone -c community -i ips` |
| SNMP data | `snmpwalk -c <community> -v1 <IP> [OID]` |

---

# Mini cheat sheet - beside-the-machine version

```bash
# 1) Quick/default TCP scan
nmap <IP>

# 2) Full TCP range - do this even if the first scan looks interesting
nmap -p 1-65535 <IP>

# 3) Service/version detection on discovered ports
nmap -sV -p <ports> <IP>

# 4) Aggressive follow-up when useful
nmap -A -p <ports> <IP>

# 5) UDP - do not forget it
sudo nmap -sU --open <IP>

# 6) Host discovery on a reachable subnet
nmap -sn <subnet/CIDR>

# 7) DNS basics
host <hostname>
host -t mx <domain>
host -t txt <domain>

# 8) DNS brute force
dnsrecon -d <domain> -D <wordlist> -t brt

# 9) Windows/internal DNS when living off the land
nslookup <hostname>

# 10) Quick single-port test from Windows
Test-NetConnection -Port <port> <IP>

# 11) SMB discovery
nmap -p 139,445 --script smb-os-discovery <IP>

# 12) Windows SMB shares
net view \\<host> /all

# 13) SMTP manual user check
nc -nv <IP> 25
# then: VRFY <username>

# 14) Find SNMP
sudo nmap -sU --open -p 161 <range>

# 15) Test SNMP communities
onesixtyone -c community -i ips

# 16) SNMP users
snmpwalk -c public -v1 <IP> 1.3.6.1.4.1.77.1.2.25

# 17) SNMP processes
snmpwalk -c public -v1 <IP> 1.3.6.1.2.1.25.4.2.1.2

# 18) SNMP local TCP ports
snmpwalk -c public -v1 <IP> 1.3.6.1.2.1.6.13.1.3
```

---

# Exam-oriented mental workflow

```text
TARGET IP
  |
  +--> TCP default scan
  |      |
  |      +--> enumerate discovered services immediately
  |
  +--> full TCP scan in parallel
  |
  +--> UDP scan
  |
  +--> Hostname/domain discovered?
  |      |
  |      +--> DNS queries / /etc/hosts / vhost testing later
  |
  +--> SMB (139/445)?
  |      |
  |      +--> host/domain/share/user enumeration
  |
  +--> SMTP (25)?
  |      |
  |      +--> banner + VRFY/EXPN if supported
  |
  +--> SNMP (161/udp)?
  |      |
  |      +--> community -> users/processes/software/local ports
  |
  +--> Exact service versions
         |
         +--> vulnerability/configuration research -> Initial Access

AFTER FOOTHOLD:
  repeat discovery from the target's point of view
  -> DNS -> internal hosts -> Test-NetConnection/port checks -> SMB/AD clues -> pivot
```

---

# Final chapter takeaway

```
# Setup workspace
mkdir -p ~/exam/{ad,box1,box2,box3}/{scans,loot,exploits,screenshots}

# Parallel quick scans on ALL targets:
for ip in <MS01_IP> <MS02_IP> <DC_IP> <BOX1_IP> <BOX2_IP> <BOX3_IP>; do
    nmap -sV --open -T4 $ip -oN ~/exam/scans/quick_$ip.txt &
done; wait

- `-sV` → detect service/version information
- `--open` → only display open ports
- `-T4` → faster/aggressive scan timing
  
# Full TCP per machine:
nmap -p- -sV -sC --open -T4 <TARGET_IP> -oN full_tcp.txt

- `-sC` → run Nmap's default NSE scripts
  
# UDP top 20 — DON'T SKIP (SNMP/TFTP/DNS often critical):
nmap -sU --top-ports 20 <TARGET_IP> -oN udp.txt
```

# OSCP Enumeration Phases and Tools

| Phase                                        | Sample Tool            | Example                                         | What You Want                                          |
| -------------------------------------------- | ---------------------- | ----------------------------------------------- | ------------------------------------------------------ |
| **1. Quick TCP scan**                        | `nmap`                 | `nmap <IP>`                                     | Quickly find common open TCP ports                     |
| **2. Full TCP scan**                         | `nmap`                 | `nmap -p- <IP>`                                 | Find services on unusual/high ports                    |
| **3. Service/version detection**             | `nmap`                 | `nmap -sV -p <ports> <IP>`                      | Exact service/product/version                          |
| **4. UDP scan**                              | `nmap`                 | `sudo nmap -sU --top-ports 20 <IP>`             | Find UDP services such as SNMP/DNS                     |
| **5. DNS / hostname discovered**             | `host`                 | `host server.domain.local`                      | Resolve hostname → IP                                  |
| **6. DNS deeper enumeration**                | `dnsrecon`             | `dnsrecon -d domain.com -t std`                 | Records, hosts, DNS information                        |
| **7. SMB 139/445**                           | Nmap NSE               | `nmap -p139,445 --script smb-os-discovery <IP>` | Computer/domain/forest/OS clues                        |
| **8. NetBIOS discovery**                     | `nbtscan`              | `sudo nbtscan -r <subnet>`                      | Windows/NetBIOS hostnames                              |
| **9. SMTP 25**                               | `nc`                   | `nc -nv <IP> 25`                                | Banner + manually try `VRFY user`                      |
| **10. SNMP 161/UDP**                         | `onesixtyone`          | `onesixtyone -c community -i ips`               | Find valid SNMP community string                       |
| **11. SNMP deep enumeration**                | `snmpwalk`             | `snmpwalk -c public -v1 <IP>`                   | Users, processes, software, ports                      |
| **12. Vulnerability research**               | Start with `nmap -sV`  | `nmap -sV -p <ports> <IP>`                      | Get accurate version first, then research that version |
| **13. After foothold — DNS**                 | `nslookup`             | `nslookup hostname`                             | Discover internal hosts/DNS                            |
| **14. After foothold — port test (windows)** | `Test-NetConnection`   | `Test-NetConnection -Port 445 <IP>`             | Test whether an internal TCP service is reachable      |
| **15. After foothold — SMB (windows)**       | `net view`             | `net view \\dc01 /all`                          | Discover Windows shares                                |
| **16. Internal port scan**                   | PowerShell `TcpClient` | `1..1024 \| % {...TcpClient...}`                | Scan internally when Nmap isn't available              |