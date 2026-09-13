# DNS enumeration (53/TCP and UDP)

## Establish the namespace

```bash
nmap -sU -sV -p53 --script dns-recursion,dns-nsid <TARGET_IP> -oN dns-udp.txt
nmap -sT -sV -p53 <TARGET_IP> -oN dns-tcp.txt
dig @<TARGET_IP> -x <TARGET_IP>
dig @<TARGET_IP> <DOMAIN> ANY
```

Collect domains and hostnames from PTR records, certificates, web redirects, SMTP banners, SMB, and SNMP; add confirmed names to `/etc/hosts` when local resolution is required.

## PEN-200 record checks with `host`

```bash
# A record (default)
host <HOST>.<DOMAIN>

# Mail and TXT records
host -t mx <DOMAIN>
host -t txt <DOMAIN>
```

Useful record types to remember:

```text
NS     authoritative name servers
A      IPv4 address
AAAA   IPv6 address
MX     mail servers
PTR    reverse lookup: IP → hostname
CNAME  alias
TXT    arbitrary text / verification / policy data
```

## Zone and name discovery

```bash
dig AXFR @<TARGET_IP> <DOMAIN>
dig @<TARGET_IP> <NAME>.<DOMAIN> A
dnsrecon -d <DOMAIN> -n <TARGET_IP> -t std,axfr
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://<TARGET_IP>/ -H 'Host: FUZZ.<DOMAIN>' -fs <BASELINE_SIZE>
```

Check TCP as well as UDP, recursion policy, zone transfers, internal records, service records (`_ldap._tcp`, `_kerberos._tcp`), and virtual hosts. Filter wildcard/baseline responses before trusting brute-force results.

## Forward DNS brute force

PEN-200 emphasizes trying likely hostnames, then feeding every positive result back into enumeration.

```bash
for name in $(cat subdomains.txt); do
    host $name.<DOMAIN>
done
```

SecLists provides larger hostname/subdomain wordlists.

### `dnsrecon` dictionary brute force

```bash
dnsrecon -d <DOMAIN> -D <WORDLIST> -t brt
```

### `dnsenum`

```bash
dnsenum <DOMAIN>
```

`dnsenum` can combine normal DNS queries, brute forcing, netrange discovery, and reverse lookups.

## Reverse DNS sweep

When forward lookups reveal several addresses in the same range, check nearby PTR records:

```bash
for ip in $(seq 1 254); do
    host <A.B.C>.$ip
done | grep -v "not found"
```

A PTR result may expose descriptive names such as `admin`, `fs1`, `intranet`, `vpn`, `snmp`, or `syslog`. Use those names to drive another round of active/passive enumeration.

## Iterative DNS workflow

```text
Known domain / hostname
        ↓
A / MX / TXT / NS queries
        ↓
Forward brute force → new hosts/IPs
        ↓
Clustered IP range? → PTR / reverse sweep
        ↓
New hostnames / roles
        ↓
Repeat DNS + port/service enumeration on every new target
```

From a Windows-only host, see [[Enumeration-stand-alone/Windows LOTL Enumeration|Windows LOTL Enumeration]] for `nslookup` examples.
