# Socat — Simple TCP Port Forwarding

Use Socat when you already control a Linux/*NIX host that can reach an internal service and you need a **simple fixed TCP forward**.

## Generic pattern

```bash
socat -ddd TCP-LISTEN:<LISTEN_PORT>,fork TCP:<DEST_IP>:<DEST_PORT>
```

Example — listen on pivot TCP/2345 and forward to internal PostgreSQL TCP/5432:

```bash
socat -ddd TCP-LISTEN:2345,fork TCP:10.4.50.215:5432
```

From Kali, connect to the pivot's listener as if it were the internal service:

```bash
psql -h <PIVOT_IP> -p 2345 -U postgres
```

Another example — expose internal SSH:

```bash
socat TCP-LISTEN:2222,fork TCP:10.4.50.215:22
ssh <USER>@<PIVOT_IP> -p 2222
```

## What the options mean

- `TCP-LISTEN:<PORT>` — open a TCP listening socket on the pivot.
- `fork` — create a child process for each connection so the forward stays available for additional connections.
- `TCP:<IP>:<PORT>` — final destination reachable from the pivot.
- `-ddd` — verbose diagnostics; useful while troubleshooting.

## OSCP notes

- A low-privileged Linux user cannot normally bind privileged ports below 1024, so choose a high listener port such as `2222`, `2345`, or `8080`.
- Socat is **not normally installed by default**. If policy/lab rules permit, a statically linked Socat binary can be transferred and run without installation.
- Verify the listener with `ss -ntplu`.
- This is a fixed one-service forward. If you need to enumerate many hosts/ports, prefer Ligolo-ng, SSH dynamic forwarding, or sshuttle.

## Other Linux forwarding fallbacks from PEN-200

- `rinetd` — daemon-oriented, more suitable for longer-lived forwarding.
- Netcat + a FIFO named pipe can be combined into a basic relay.
- With root, `iptables` can create forwards, but forwarding may first need to be enabled for the interface, e.g. `/proc/sys/net/ipv4/conf/<interface>/forwarding`.
