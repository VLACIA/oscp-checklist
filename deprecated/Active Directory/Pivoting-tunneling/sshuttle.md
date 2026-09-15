# sshuttle — SSH as a VPN-like Pivot

`sshuttle` is useful when you have **direct SSH access to a server/pivot behind which there are multiple internal subnets**. It creates local routes and transparently sends matching traffic through SSH, so most tools can access internal IPs normally without a `proxychains` prefix.

## Requirements / trade-offs

- `sshuttle` runs on the **SSH client** (normally Kali).
- It requires **root privileges on the SSH client** to manipulate local routing/firewall state.
- It requires **Python 3 on the SSH server**.
- It is convenient for complex internal networks, but heavier than a single `-L` or `-R` forward.

## Basic syntax

```bash
sudo sshuttle -r <USER>@<SSH_SERVER> <SUBNET_1> <SUBNET_2>
```

Example:

```bash
sudo sshuttle -r database_admin@<PIVOT_IP>:2222 10.4.50.0/24 172.16.50.0/24
```

After it connects, interact with internal systems directly:

```bash
smbclient -L //172.16.50.217/ -U <USER>
nmap -sT -Pn -n 172.16.50.217
```

## When the SSH server itself is reached through a forward

You can combine a simple forward with sshuttle. For example, if a first pivot exposes an internal SSH server on `<PIVOT_IP>:2222`, point sshuttle at that forwarded SSH endpoint:

```bash
sudo sshuttle -r <USER>@<PIVOT_IP>:2222 10.4.50.0/24 172.16.50.0/24
```

Use this when you want route-like access and the requirements are already satisfied. If you cannot meet the Python/root requirements, fall back to Ligolo-ng, SSH SOCKS + Proxychains, Chisel, or fixed port forwards.
