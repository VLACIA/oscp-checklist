# RDP enumeration (3389/TCP and UDP)

## Identify the endpoint

```bash
nmap -sV -p3389 --script rdp-enum-encryption,rdp-ntlm-info <TARGET_IP> -oN rdp.txt
```

Record the DNS/computer/domain names, OS/build hints, certificate, Network Level Authentication requirement, and supported security layers.

## Validate known credentials

```bash
xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>' /cert:ignore +clipboard
# Add /d:<DOMAIN> when domain authentication is required.
```

- Try the correct local (`.\\user`) versus domain identity format.
- Treat BlueKeep or other RDP CVEs as candidates only after exact OS/build and prerequisites are confirmed; scanner output alone is not proof.
- A valid account may lack the “Allow log on through Remote Desktop Services” right.
- Avoid repeated login attempts because lockout policies apply.
