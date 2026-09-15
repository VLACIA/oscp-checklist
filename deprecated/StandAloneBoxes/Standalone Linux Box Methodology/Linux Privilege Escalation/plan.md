# Linux Privilege Escalation Checklist

> PEN-200 Chapter 17 integration: **enumerate first, exploit second**. Automated tools are useful, but they do not replace manual inspection of one-off host configurations.

## Mental model

**Privilege escalation = you already have a shell as a low-privileged user and want to become `root`.** The goal is to find a privileged action, credential, file, process, or system component that you can influence.

A practical OSCP order:

1. **Manual baseline + user context** — identity, users, hostname, OS/kernel/architecture, root-owned processes.
2. **Credential hunting** — environment variables, dotfiles/history, process arguments, service traffic, password reuse.
3. **Sudo rights** — `sudo -l`; inspect each permitted binary and its exact restrictions.
4. **SUID/SGID + capabilities** — identify unusual privileged binaries and check GTFOBins.
5. **Cron / writable privileged files** — find root jobs and verify permissions on every referenced script/binary/directory.
6. **Writable sensitive files** — especially `/etc/passwd` or privileged configuration/scripts.
7. **Local services / network clues** — loopback-only services, extra interfaces/routes, firewall artifacts, pivot opportunities. If you discover a network Kali cannot route to, hand off to [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]].
8. **Mounted/unmounted storage + packages/modules** — look for secrets and vulnerable software/components.
9. **Automated enumeration** — LinPEAS / `unix-privesc-check` / LinEnum to catch additional clues, then verify manually.
10. **Kernel exploit last** — match distro + kernel + architecture, inspect/compile carefully, and expect crash risk.

## First-pass questions

- Who am I? Which groups am I in? Are there other interactive users?
- What exact distro, kernel, and CPU architecture am I on?
- What is running as `root`? Do command lines expose credentials?
- Are there secrets in `env`, `.bashrc`, `.profile`, history, or configuration files?
- Does `sudo -l` expose a usable command?
- Are unusual SUID/SGID binaries or Linux capabilities present?
- Does root execute a script I can modify through cron?
- Are sensitive files or privileged script directories writable?
- Are there localhost-only services, extra NICs/routes, or readable firewall rules?
- Are there interesting mounted/unmounted disks, packages, drivers, or kernel modules?

## Important reminders

- **Do not rely only on LinPEAS.** Custom misconfigurations are exactly what automated tools may miss.
- A recovered password is not just one credential: test **credential reuse** against other local users and permitted services.
- A promising GTFOBins technique may still be blocked by controls such as **AppArmor**. If an apparently valid technique fails, inspect logs before abandoning the path.
- Treat kernel exploits as a **late option**. A mismatched exploit can destabilize or crash the host.

See:
- [[Manual Enumeration and Credential Hunting]]
- [[Commands|Linux PrivEsc Commands]]
- [[Decision Tree|Linux PrivEsc Decision Tree]]


## After root / before leaving the host

Re-run the network checks with full privileges:

```bash
ip addr
ip route
ss -ntplu
```

A newly visible interface, route, firewall rule, or internal-only listener can turn the rooted Linux host into a pivot. If Kali cannot directly reach the discovered network/service, continue with [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]] rather than trying to force every tool to run on the compromised host.
