# Chain G — Anonymous FTP Write + Webroot Overlap → Shell

**Attack path:** anonymous FTP → writable web directory → upload executable script → request script over HTTP → shell as `www-data` → privilege escalation

## Brief explanation

1. **Check anonymous FTP access:** Misconfigured FTP servers may allow anonymous users to log in and upload files.
2. **Map FTP storage to the web server:** A writable FTP directory may correspond to a web-served path such as `/var/www/html/uploads`.
3. **Upload and request a test file:** Confirm the mapping with harmless content first. If the directory also permits server-side script execution, an uploaded PHP file can provide command execution.
4. **Enumerate local escalation paths:** From the web-service account, inspect Linux capabilities, SUID files, cron jobs, `sudo` rules, and writable privileged scripts.

**Why the chain works:** Two individually risky configurations overlap: unauthenticated users can write files, and the web server executes files from that same location.
