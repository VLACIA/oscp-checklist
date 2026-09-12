```
# Local port forward:
ssh -L 3389:10.10.10.50:3389 user@<MS01_IP>
xfreerdp /v:localhost:3389 /u:user /p:pass

# Dynamic SOCKS:
ssh -D 1080 -N user@<MS01_IP>
proxychains nmap -sT -Pn 10.10.10.0/24
```