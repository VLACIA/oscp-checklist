# TCP / UDP Port Scanning Theory

> **PEN-200 Chapter 6 — §6.3.2–6.3.3.** Port scanning is an iterative discovery process. The result of one scan should determine the scope and type of the next scan rather than blindly launching the heaviest scan first.

## TCP CONNECT scanning

A normal TCP connection completes the three-way handshake:

```text
Client  → SYN     → Server
Client  ← SYN-ACK ← Server
Client  → ACK     → Server
```

A successful handshake indicates an open TCP port.

### Netcat example

```bash
nc -nvv -w 1 -z <TARGET_IP> 1-1024
```

Useful options from PEN-200:

```text
-n  numeric IPs; no DNS resolution
-vv verbose
-w  timeout
-z  zero-I/O scanning mode
```

## SYN scan (`-sS`)

```bash
sudo nmap -sS <TARGET_IP>
```

- Sends SYN probes but does not complete the full TCP handshake.
- An open port normally replies with SYN-ACK.
- Faster and uses fewer packets than a full connect scan.
- Requires raw-socket privileges.
- The historical name **stealth scan** is misleading against modern firewalls; incomplete connections can still be detected/logged.

## TCP connect scan (`-sT`)

```bash
nmap -sT <TARGET_IP>
```

- Completes the TCP handshake through the normal socket API.
- Does not require raw-socket privileges.
- Slower than SYN scanning.
- Can be useful when raw sockets are unavailable or in some proxy-based situations.

## UDP scanning

UDP has no three-way handshake.

```bash
nc -nv -u -z -w 1 <TARGET_IP> 1-1024
sudo nmap -sU <TARGET_IP>
```

Typical interpretation:

```text
Closed UDP port → target may return ICMP Port Unreachable
Open UDP port   → application may reply, or may remain silent
Filtered path   → firewall may suppress ICMP, creating ambiguity
```

### Important UDP pitfall

Silence does **not** reliably prove a UDP port is open. Firewalls/routers may drop ICMP responses, so UDP scanning can produce false positives or `open|filtered` results. Recheck important UDP ports with protocol-specific probes/scripts.

Do not forget UDP: useful attack surface can exist on services such as DNS and SNMP.

## Network sweeping

```bash
nmap -sn <CIDR>
nmap -v -sn <CIDR> -oG ping-sweep.txt
grep Up ping-sweep.txt | cut -d " " -f 2
```

PEN-200 notes that `-sn` host discovery is more than a simple ICMP echo check; Nmap can use several probes to determine whether hosts are up.

A service-specific sweep can be more useful than a generic ping sweep:

```bash
nmap -p 80 <CIDR> -oG web-sweep.txt
grep open web-sweep.txt | cut -d " " -f 2
```

## Nmap fingerprinting and service enumeration

```bash
# OS fingerprinting / force a best-effort guess
sudo nmap -O <TARGET_IP> --osscan-guess

# Service/version detection
nmap -sV -p<PORTS> <TARGET_IP>

# Aggressive bundle: OS + versions + scripts + traceroute
nmap -A -p<PORTS> <TARGET_IP>
```

Treat OS and service detection as **evidence, not certainty**. Firewalls/proxies can alter packet characteristics and banners can be intentionally changed.

## NSE scripts

```bash
ls -1 /usr/share/nmap/scripts/
nmap --script-help <SCRIPT_NAME>
nmap --script <SCRIPT_NAME> -p<PORT> <TARGET_IP>
```

NSE scripts can automate discovery, enumeration, brute-force checks, and vulnerability identification. Prefer targeted scripts against services you have already discovered. Before running vulnerability scripts, check whether they are categorized `safe`, `intrusive`, or `exploit`; intrusive checks may affect target stability.

For `--script "vuln"`, `vulners`, custom NSE scripts, false positives/negatives, and scanner limitations, see [[Enumeration-stand-alone/Vulnerability Scanning|Vulnerability Scanning]].

## OSCP scanning mindset

```text
1. Discover broadly.
2. Save output.
3. Identify hosts/services worth deeper attention.
4. Run targeted version/default-script scans.
5. Re-scan ambiguous UDP or filtered results with protocol-specific probes.
6. Feed new hostnames, ports, versions, and credentials back into enumeration.
```

See also: [[Enumeration-stand-alone/nmap|Nmap workflow]].
