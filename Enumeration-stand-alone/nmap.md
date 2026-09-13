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

tag:#enumeration
