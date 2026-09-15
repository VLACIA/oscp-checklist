### Initial Recon — Run in Parallel

```
# Setup workspace
mkdir -p ~/exam/{ad,box1,box2,box3}/{scans,loot,exploits,screenshots}

# Parallel quick scans on ALL targets:
for ip in <MS01_IP> <MS02_IP> <DC_IP> <BOX1_IP> <BOX2_IP> <BOX3_IP>; do
    nmap -sV --open -T4 $ip -oN ~/exam/scans/quick_$ip.txt &
done; wait

# 1. Full TCP discovery. Do not attach heavy scripts to all 65,535 ports.
sudo nmap -p- --open -T4 --min-rate 1000 <TARGET_IP> -oA full_tcp

# 2. Targeted UDP discovery (expand beyond this when clues justify it).
sudo nmap -sU --top-ports 100 --open -T4 <TARGET_IP> -oA udp_top100

# 3. Version and default scripts against the ports actually discovered.
sudo nmap -sV -sC -p<TCP_PORTS> <TARGET_IP> -oA targeted_tcp
sudo nmap -sU -sV -sC -p<UDP_PORTS> <TARGET_IP> -oA targeted_udp
```

If a host appears down, retry discovery with `-Pn`. Keep TCP and UDP results separate, record ambiguous `open|filtered` UDP ports, and rescan important ports with protocol-specific scripts. Parse the full scan output into a comma-separated port list before the targeted scan.

## PEN-200 scan variants / fallbacks

```bash
# SYN scan — raw socket privileges required
sudo nmap -sS <TARGET_IP>

# TCP connect scan — no raw socket privileges required
nmap -sT <TARGET_IP>

# Network sweep
nmap -sn <CIDR> -oG ping-sweep.txt
grep Up ping-sweep.txt | cut -d " " -f 2

# Sweep for a specific service
nmap -p80 <CIDR> -oG web-sweep.txt

# OS fingerprinting / best-effort guess
sudo nmap -O <TARGET_IP> --osscan-guess

# Inspect NSE script documentation
nmap --script-help <SCRIPT_NAME>
```

Treat OS fingerprints, service versions, and banners as clues rather than guaranteed truth. See [[Enumeration-stand-alone/Port Scanning Theory|TCP / UDP Port Scanning Theory]] for SYN vs connect behavior, UDP ambiguity, network sweeping, and NSE notes.


## PEN-200 Chapter 7 — NSE vulnerability scanning

After accurate service/version detection, Nmap can be used as a **lightweight vulnerability scanner** with NSE. Treat results as candidates that still require manual verification.

```bash
# List scripts in the vuln category
cd /usr/share/nmap/scripts/
cat script.db | grep '"vuln"'

# Run vulnerability scripts against known ports/services
sudo nmap -sV -p <PORTS> --script "vuln" <TARGET_IP>

# After adding a reviewed third-party/custom NSE script
sudo nmap --script-updatedb
sudo nmap -sV -p <PORT> --script "<SCRIPT_NAME>" <TARGET_IP>
```

Before running NSE checks, inspect whether the script is categorized **safe**, **intrusive**, or **exploit**. Intrusive scripts may affect target stability, and third-party scripts must be reviewed before execution.

The integrated `vulners` script can map detected service versions to CVEs/CVSS data and may list PoCs marked `*EXPLOIT*`, but it depends on successful service/version detection.

Full workflow and scanner pitfalls: [[Enumeration-stand-alone/Vulnerability Scanning|Vulnerability Scanning]].


## PEN-200 Chapter 15 — find NSE exploit scripts

NSE is not only for enumeration and `vuln` checks; Kali also ships scripts that can actively exploit specific products. Search the installed scripts, then read the script help before using one:

```bash
# Find installed NSE scripts whose description says they exploit something
grep Exploits /usr/share/nmap/scripts/*.nse

# Read one script's purpose, affected versions, references, and arguments
nmap --script-help=<SCRIPT>.nse
```

If you identify an exact product/version during enumeration, check whether a matching NSE script exists. Treat scripts in the **exploit** or **intrusive** categories as potentially disruptive and review them before execution.

tag:#enumeration
