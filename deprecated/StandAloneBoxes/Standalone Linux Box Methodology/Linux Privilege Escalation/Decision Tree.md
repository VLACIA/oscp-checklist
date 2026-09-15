# Linux PrivEsc Decision Tree

```text
FOOTHOLD
│
├─ 1. Manual baseline
│    ├─ id / users / hostname
│    ├─ distro + kernel + architecture
│    ├─ root-owned processes
│    ├─ interfaces / routes / listening ports
│    ├─ cron / writable files / mounts / packages / modules
│    └─ credentials in env, dotfiles, history, process argv, traffic
│
├─ Recovered credential?
│    ├─ Try root / other local users / known services
│    └─ New user shell → run `sudo -l` again
│
├─ `sudo -l` useful?
│    ├─ GTFOBins / allowed behavior → ROOT ✓
│    └─ Technique blocked? → inspect syslog/AppArmor/audit → try next path
│
├─ SUID / SGID binary?
│    ├─ Known binary → GTFOBins → preserve effective UID where required → ROOT ✓
│    └─ Custom binary → inspect strings/ltrace/config/PATH behavior → ROOT path?
│
├─ Linux capability?
│    └─ Dangerous capability such as `cap_setuid+ep` → GTFOBins → ROOT ✓
│
├─ Root cron job / privileged script?
│    ├─ Script or dependent path writable → modify payload → wait for execution → ROOT ✓
│    └─ Not writable → continue
│
├─ `/etc/passwd` writable?
│    └─ Add UID/GID 0 account with valid password hash → `su` → ROOT ✓
│
├─ Local-only privileged service / readable firewall clue?
│    └─ Interact locally / forward port / exploit service → ROOT path?
│
├─ Interesting disk/partition or configuration?
│    └─ Search for backups, credentials, scripts, keys → reuse / escalate
│
├─ NFS `no_root_squash`?
│    └─ Mount + create SUID executable → ROOT ✓
│
├─ Docker group?
│    └─ Bind-mount `/` + chroot → ROOT ✓
│
├─ Vulnerable userland component? (existing vault examples)
│    └─ Exact-version PoC (e.g. applicable glibc/pkexec/sudo flaw) → ROOT ✓
│
├─ Automated enumeration reveals missed misconfiguration?
│    └─ LinPEAS / unix-privesc-check / LinEnum → manually validate finding
│
└─ Nothing simpler?
     └─ KERNEL EXPLOIT LAST
          ├─ exact distro + kernel + architecture
          ├─ SearchSploit → read exploit requirements
          ├─ compile for target architecture (prefer target when gcc exists)
          ├─ verify binary with `file`
          └─ execute carefully → ROOT ✓
```

**Rule:** prefer credentials and configuration mistakes over kernel exploits. They are usually easier to validate and less likely to crash the target.
