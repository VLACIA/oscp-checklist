### Port → Action Decision Table

| Port      | Service  | Immediate Action                                                 |
| --------- | -------- | ---------------------------------------------------------------- |
| 21        | FTP      | ftp <IP> (try anonymous:anonymous); check version → searchsploit |
| 22        | SSH      | Note version; use creds found later; id_rsa files?               |
| 25        | SMTP     | smtp-user-enum; check open relay                                 |
| 53        | DNS      | dig axfr @<IP> <domain>; dnsrecon zone transfer                  |
| 80/443    | HTTP/S   | → Full web enumeration workflow below                            |
| 88        | Kerberos | AD confirmed! AS-REP roast immediately                           |
| 111       | RPC/NFS  | showmount -e <IP>; mount + explore                               |
| 139/445   | SMB      | → SMB enumeration section below                                  |
| 161 UDP   | SNMP     | snmpwalk -c public -v1; onesixtyone brute                        |
| 389/636   | LDAP     | ldapsearch anonymous; AD enumeration                             |
| 1433      | MSSQL    | → Full MSSQL section (Section 6)                                 |
| 3306      | MySQL    | mysql -u root -h <IP> (blank/default pass)                       |
| 3389      | RDP      | Check ms17-010, BlueKeep; use creds                              |
| 5985      | WinRM    | evil-winrm -i <IP> -u user -p pass                               |
| 8080/8443 | Alt-HTTP | Tomcat? Jenkins? Weblogic? check /manager                        |
tag:#enumeration 