# SNMP enumeration (161/UDP)

```bash
# Find exposed SNMP
sudo nmap -sU --open -p161 <CIDR> -oG open-snmp.txt

# Community-string discovery
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>

# Walk the accessible MIB tree
snmpwalk -c public -v1 -t 10 <TARGET_IP>

# Running processes
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.2

# Process paths
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.4

# Installed software
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2

# Windows users
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.4.1.77.1.2.25

# Local listening TCP ports — can reveal services hidden from remote scans
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.6.13.1.3

snmp-check <TARGET_IP> -c public
```

Try the SNMP version actually supported by the target (`-v1` or `-v2c`). SNMP v1/v2/v2c do not encrypt traffic and weak/default community strings are a common source of information disclosure.

Cross-check running programs, executable paths, and installed software to identify interesting application versions. Local TCP-port enumeration may disclose services listening only locally.

See [[Enumeration-stand-alone/SNMP OID Reference|SNMP OID Reference]].
