# DNS enumeration (53/TCP and UDP)

## Establish the namespace

```bash
nmap -sU -sV -p53 --script dns-recursion,dns-nsid <TARGET_IP> -oN dns-udp.txt
nmap -sT -sV -p53 <TARGET_IP> -oN dns-tcp.txt
dig @<TARGET_IP> -x <TARGET_IP>
dig @<TARGET_IP> <DOMAIN> ANY
```

Collect domains and hostnames from PTR records, certificates, web redirects, SMTP banners, SMB, and SNMP; add confirmed names to `/etc/hosts` when local resolution is required.

## Zone and name discovery

```bash
dig AXFR @<TARGET_IP> <DOMAIN>
dig @<TARGET_IP> <NAME>.<DOMAIN> A
dnsrecon -d <DOMAIN> -n <TARGET_IP> -t std,axfr
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://<TARGET_IP>/ -H 'Host: FUZZ.<DOMAIN>' -fs <BASELINE_SIZE>
```

Check TCP as well as UDP, recursion policy, zone transfers, internal records, service records (`_ldap._tcp`, `_kerberos._tcp`), and virtual hosts. Filter wildcard/baseline responses before trusting brute-force results.
