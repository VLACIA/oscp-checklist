# SNMP OID Reference

> **PEN-200 Chapter 6 — §6.3.6.** SNMP can disclose far more than network-device metadata. Weak/default community strings can expose users, processes, software, paths, and even local listening ports.

## Why SNMP matters

- SNMP uses UDP.
- SNMP v1/v2/v2c do not encrypt traffic.
- Weak/default community strings such as `public` and `private` are common misconfigurations.
- The Management Information Base (MIB) is a tree of values; an OID identifies a branch/value to query.

## Find SNMP

```bash
sudo nmap -sU --open -p161 <CIDR> -oG open-snmp.txt
```

## Try community strings

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>
```

PEN-200 also demonstrates creating a small community list (`public`, `private`, `manager`) and scanning a list of IP addresses with `onesixtyone`.

## Walk the accessible MIB tree

```bash
snmpwalk -c public -v1 -t 10 <TARGET_IP>
```

Replace the version/community string with the values that the target actually supports.

## Useful Windows OIDs from PEN-200

| Information | OID |
|---|---|
| System process count | `1.3.6.1.2.1.25.1.6.0` |
| Running programs | `1.3.6.1.2.1.25.4.2.1.2` |
| Process executable paths | `1.3.6.1.2.1.25.4.2.1.4` |
| Storage units | `1.3.6.1.2.1.25.2.3.1.4` |
| Installed software names | `1.3.6.1.2.1.25.6.3.1.2` |
| Windows user accounts | `1.3.6.1.4.1.77.1.2.25` |
| Local TCP listening ports | `1.3.6.1.2.1.6.13.1.3` |

## Direct queries

```bash
# Windows users
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.4.1.77.1.2.25

# Running programs
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.2

# Process paths
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.4

# Installed software
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2

# Local listening TCP ports
snmpwalk -c public -v1 <TARGET_IP> 1.3.6.1.2.1.6.13.1.3
```

### Especially useful: local TCP ports

The TCP-port OID can reveal services listening only on localhost/internal interfaces that were invisible to your remote Nmap scan. Use that information to investigate local services after foothold or to identify pivoting opportunities.

Cross-check **running process + executable path + installed software** to identify the exact application/version behind an interesting process.

See also: [[Enumeration-stand-alone/snmp enum|SNMP enumeration]].
