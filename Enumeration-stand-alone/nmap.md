### Initial Recon — Run in Parallel

```
# Setup workspace
mkdir -p ~/exam/{ad,box1,box2,box3}/{scans,loot,exploits,screenshots}

# Parallel quick scans on ALL targets:
for ip in <MS01_IP> <MS02_IP> <DC_IP> <BOX1_IP> <BOX2_IP> <BOX3_IP>; do
    nmap -sV --open -T4 $ip -oN ~/exam/scans/quick_$ip.txt &
done; wait

# Full TCP per machine:
nmap -p- -sV -sC --open -T4 <TARGET_IP> -oN full_tcp.txt

# UDP top 20 — DON'T SKIP (SNMP/TFTP/DNS often critical):
nmap -sU --top-ports 20 <TARGET_IP> -oN udp.txt
```

tag:#enumeration 