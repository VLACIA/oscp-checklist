# Pivoting and Tunneling

**Pivoting = using a compromised machine as a stepping stone to reach systems or networks Kali cannot route to directly.** The pivot host can talk to both sides, so it becomes the bridge for your traffic.

**Simple analogy:** you cannot enter a locked office building, but someone inside can pass your messages back and forth. That insider is your pivot.

For OSCP, **Ligolo-ng is still the preferred first choice** when you can transfer and run an agent because it gives Kali route-like access to the internal network. Keep SSH, Socat, sshuttle, Chisel, Plink, and Netsh as situation-dependent backups.

## When to think "pivot"

After every foothold and again after privilege escalation, check the host's network view:

```bash
# Linux
ip addr
ip route
ss -ntplu
```

```powershell
# Windows
ipconfig /all
route print
netstat -ano
```

Look for:
- an extra NIC or subnet that Kali cannot reach directly;
- localhost/internal-only services that were invisible from Kali;
- firewall rules that block inbound access but still permit outbound connections.

## OSCP decision workflow

```text
New subnet / internal service found?
        |
        +-- Kali can already route to it --> enumerate normally
        |
        +-- Kali cannot route to it --> choose a pivot
                |
                +-- Can run Ligolo agent? --> Ligolo-ng (preferred)
                |
                +-- SSH access to a pivot/server?
                |       +-- one fixed service --> SSH -L
                |       +-- many hosts/ports  --> SSH -D + Proxychains
                |
                +-- Inbound to pivot blocked, but pivot can SSH OUT to Kali?
                |       +-- one fixed service --> SSH -R host:port:target:port
                |       +-- many hosts/ports  --> SSH -R <port> (remote dynamic)
                |
                +-- Direct SSH server + complex internal routes? --> sshuttle
                |
                +-- Need only a simple fixed TCP forward on Linux? --> Socat
                |
                +-- Windows pivot? --> ssh.exe / Plink / Netsh
                |
                +-- Normal tunnel traffic blocked / suspect DPI?
                        +-- HTTP/WebSocket outbound allowed --> Chisel reverse SOCKS
                        +-- DNS queries still escape via resolver --> dnscat2 DNS tunnel
```

## Restricted egress / DPI decision

A firewall may allow/deny traffic only by IP and port, while **Deep Packet Inspection (DPI)** can inspect the protocol itself. If SSH transport is explicitly blocked, SSH `-L`, `-D`, and `-R` tunnels may all fail even if you try a normally allowed TCP port.

When a normal pivot fails, test **what outbound protocol is actually permitted**:

```text
Normal pivot fails
      |
      +-- Pivot can make HTTP/WebSocket requests to Kali?
      |       --> [[Chisel|Chisel reverse SOCKS over HTTP]]
      |
      +-- Pivot cannot connect out directly, but DNS lookups work?
              --> [[DNS Tunneling - dnscat2|dnscat2 through DNS]]
```

Do not jump to DNS tunneling just because a port is closed. First confirm routing, local firewalls, listener configuration, and whether the pivot itself can reach the intended target.

## Important mental model

- **Port forwarding** changes where traffic is sent: a listening socket relays traffic to another socket.
- **Tunneling** encapsulates one traffic stream inside another; SSH can carry other protocol traffic inside its encrypted connection.
- For SSH, always ask **where is the listening port?** and **which side can reach the final target?** This determines whether you want `-L`, `-D`, or `-R`.

## Notes

- Prefer ports **above 1024** when the compromised Linux account is not root.
- Verify listeners after creating them (`ss -ntplu`, `netstat -ano`).
- With SOCKS pivots, use tools that support SOCKS or wrap them with Proxychains.
- When the tunnel fails, confirm routing from the pivot itself before troubleshooting the tunnel.

See:
- [[Ligolo-ng]]
- [[SSH Tunneling]]
- [[Socat]]
- [[sshuttle]]
- [[Chisel]]
- [[Plink (Windows — no SSH client)]]
- [[Netsh (Windows native port forwarding)]]

- [[DNS Tunneling - dnscat2]]

## Metasploit option

If you already have a Meterpreter session and want an MSF-native pivot, see [[Basics/Metasploit/Metasploit Pivoting|Metasploit Pivoting]] for `route` / `autoroute`, SOCKS + Proxychains, bind-payload considerations, and `portfwd`.
