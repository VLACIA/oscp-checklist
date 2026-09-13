tag:#enumeration 

**Immediately after Nmap finds HTTP/HTTPS** on 80, 443, 8000, 8080, 8443, etc.

```
# 1. Tech fingerprint
whatweb http://<TARGET_IP> -v
curl -I http://<TARGET_IP>
nikto -h http://<TARGET_IP> -output nikto.txt &

# 2. Directory brute force
gobuster dir -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
    -x php,html,txt,asp,aspx,jsp -t 50 -o gobuster.txt

feroxbuster -u http://<TARGET_IP> \
    -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
    -x php,html,txt,aspx -t 100 --depth 3 -o ferox.txt

# 3. Always manually check:
curl http://<TARGET_IP>/robots.txt
curl http://<TARGET_IP>/.git/HEAD
curl http://<TARGET_IP>/.env
curl http://<TARGET_IP>/backup/
curl http://<TARGET_IP>/config.php
view-source:http://<TARGET_IP>/           # HTML comments!

# 4. VHost enumeration:
gobuster vhost -u http://<DOMAIN> \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    --append-domain -t 50 -o vhosts.txt

# 5. CMS-specific:
wpscan --url http://<TARGET_IP> --enumerate u,p,t,cb,dbe --plugins-detection aggressive
joomscan -u http://<TARGET_IP>

# 6. Parameter fuzzing:
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
    -u "http://<TARGET_IP>/page.php?FUZZ=test" -fs 0 -mc 200,301,302,500
```


