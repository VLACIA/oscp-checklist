**Key Insight:** OSCP boxes rarely have a single vulnerability. They chain 2-3 misconfigs. These are the most common patterns found in real exam/PG boxes.

**Summary:**

Chain A — Web Version CVE → Cred → Privesc Apache/Nginx/App version found → searchsploit → RCE as www-data → config.php has DB creds → reuse creds for SSH → sudo -l → ROOT

Chain B — Anonymous SMB → Creds → RCE SMB null session → readable share → config/pdf/xlsx file → creds found → SSH/WinRM login → SUID/SeImpersonate → ROOT

Chain C — LFI → SSH Key Read → Shell → Privesc LFI at ?file= param → /home/user/.ssh/id_rsa → chmod 600 → SSH in → sudo -l NOPASSWD → GTFObins → ROOT

Chain D — Web Login → File Upload → Webshell → Cron Root Default creds (admin:admin) → file upload → bypass (phar/phtml) → webshell as www-data → /etc/crontab writable script → ROOT

Chain E — LFI + PHP Filter → DB Creds → SQLi File Write → RCE LFI + php://filter → base64 decode config.php → DB creds → SQLi UNION INTO OUTFILE → webshell → shell → privesc

Chain F — .git Exposed → Deleted Creds → SSH → sudo /.git/HEAD accessible → git-dumper → git log/diff → password found in old commit → SSH login → sudo privesc → ROOT

Chain G — FTP Anon Write + Webroot Overlap → Shell FTP anonymous login → writable dir = /var/www/html/uploads → upload PHP shell → trigger via browser → www-data → capabilities/cron → ROOT

Chain H — SNMP Community → Username → Password Spray → RCE snmpwalk → running process with username / installed software version → username found → spray common passwords → SSH/WinRM → SeImpersonate or SUID → ROOT

Chain I — Username as Password (Very Common in OSCP) Any service leaks username (SMTP enum, SNMP, web, SMB) → try username:username, username:username123, username:Company1 → login → privesc

Chain J — SQLi → OS Shell (MSSQL/MySQL) SQLi on login form → xp_cmdshell (MSSQL) or UDF (MySQL) → RCE as service account → SeImpersonate → SYSTEM

