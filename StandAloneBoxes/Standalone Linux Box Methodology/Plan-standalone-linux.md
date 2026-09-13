# Linux Standalone Box Methodology

## How to think about a box

**A "standalone box" is one machine to fully own — get a shell, then escalate to root.** Most Linux boxes are cracked through a web app on port 80/443, so that is usually where you start. The web attacks below are common ways in: find the flaw, get code execution, land a shell.

**Core loop:** enumerate → identify an evidence-backed weakness → obtain a shell → manually enumerate locally → escalate to `root` → capture proof.

## After foothold

1. Stabilize the shell and confirm identity/context.
2. Run the Linux privilege-escalation checklist: [[Linux Privilege Escalation/plan|Linux PrivEsc — ordered checklist]].
3. Enumerate manually before relying on automation: [[Linux Privilege Escalation/Manual Enumeration and Credential Hunting]].
4. Escalate to `root`, verify `id`/hostname, and capture proof.

## After root: check whether this host is also a pivot

Before treating the machine as finished, quickly re-check its network view:

```bash
ip addr
ip route
ss -ntplu
```

If the host has an **extra interface/subnet or internal service that Kali cannot reach directly**, continue into [[Active Directory/Pivoting-tunneling/Intro|Pivoting and Tunneling]]. This is optional on a truly standalone target, but essential when the compromised Linux host bridges you into another network.
