# Netsh — Windows Native Port Forwarding

Windows can create a native TCP port forward with `netsh interface portproxy`. This is useful when you have **administrator privileges** on a Windows pivot and it can reach an internal service.

> `portproxy` changes require administrative privileges. In an interactive desktop/RDP session, UAC may also matter.

## 1. Add a v4-to-v4 forward

```cmd
netsh interface portproxy add v4tov4 listenport=<LISTEN_PORT> listenaddress=<PIVOT_IP> connectport=<DEST_PORT> connectaddress=<DEST_IP>
```

Example — Windows pivot `192.168.50.64:2222` → internal SSH `10.4.50.215:22`:

```cmd
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=192.168.50.64 connectport=22 connectaddress=10.4.50.215
```

Verify locally:

```cmd
netstat -anp TCP | find "2222"
netsh interface portproxy show all
```

## 2. Remember the Windows Firewall

A portproxy listener can exist while inbound traffic is still blocked. If Kali sees the new port as **filtered**, add an inbound firewall rule:

```cmd
netsh advfirewall firewall add rule name="port_forward_ssh_2222" protocol=TCP dir=in localip=192.168.50.64 localport=2222 action=allow
```

Recheck from Kali:

```bash
nmap -sS -Pn -n -p2222 192.168.50.64
ssh <USER>@192.168.50.64 -p2222
```

## 3. Cleanup

Remove the firewall opening:

```cmd
netsh advfirewall firewall delete rule name="port_forward_ssh_2222"
```

Remove the portproxy rule:

```cmd
netsh interface portproxy del v4tov4 listenport=2222 listenaddress=192.168.50.64
```

## Exam reminder

**Portproxy rule and firewall rule are separate.** Creating the forward does not automatically allow inbound connections through Windows Firewall. Use a descriptive firewall-rule name so you can remove it cleanly afterward.
