### Linux Privilege Escalation Checklist

Mental model

**Privilege escalation = you're already inside as a normal user; now you want to become `root`.** You're hunting for one mistake the admin made. The four most common on OSCP boxes, in order of frequency:

**1. Sudo rights** — `sudo -l` shows commands you can run as root. Look each one up on [GTFOBins](https://gtfobins.github.io/). **2. SUID binaries** — programs that run as their owner (often root); again, check GTFOBins. **3. Cron jobs** — scripts root runs on a schedule that you can edit. **4. Credentials lying around** — config files, `.bash_history`, database passwords reused for the root account.

**Always run an automated enumerator first** (`linpeas.sh`), but never _only_ rely on it — read the output and verify the RED/YELLOW findings manually. linpeas finds clues; _you_ connect them.