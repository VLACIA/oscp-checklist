tag:#basic

# Linux File Permissions

On Linux, every file says _who_ can **r**ead, **w**rite, and e**x**ecute it — split into three categories: the **owner**, the **group**, and **everyone else**. Misconfigured permissions (for example, a root-owned script that your user can modify) are a classic privilege-escalation path.

View permissions with:

```bash
ls -l
```

Example:

```text
-rwxrwxr-x 1 mkarami mkarami 5795 Sep 11 09:38 universal.ovpn
```

- First `-` = regular file
- Owner: `rwx` = read, write, execute
- Group: `rwx` = read, write, execute
- Others: `r-x` = read and execute, but cannot write

Equivalent numeric permission:

```text
775
```

because `rwx = 7`, `rwx = 7`, and `r-x = 5`.

## File vs directory permissions

The same letters mean different things for directories:

| Permission | File | Directory |
| --- | --- | --- |
| `r` | Read file contents | List directory entries |
| `w` | Modify file contents | Create/delete entries in the directory |
| `x` | Execute the file | Traverse/access items inside the directory |

A directory can be **executable/traversable without being readable**. In that case, you may access an entry if you already know its exact name even though you cannot list the directory.

## SUID / SGID

Linux also has special executable permissions:

- **SUID / setuid**: the program runs with the **effective UID of the file owner**.
- **SGID / setgid**: the program runs with the **effective GID of the file's group**.
- They appear as `s`/`S` in the execute position.

Find them:

```bash
find / -perm -4000 -type f 2>/dev/null   # SUID
find / -perm -2000 -type f 2>/dev/null   # SGID
```

Example: `/usr/bin/passwd` is normally SUID root so a regular user can update the protected `/etc/shadow` file through the constrained `passwd` program. If a SUID-root binary can be abused to run an arbitrary command, the resulting process may inherit an effective UID of root.

For a running process, Linux exposes the real/effective/saved/filesystem UIDs in `/proc`:

```bash
grep Uid /proc/<PID>/status
```

The second value is the **effective UID** used for permission checks.

The SUID bit can be set with:

```bash
chmod u+s <file>
```

During privilege escalation, prioritize writable files/directories that are consumed by **root-owned processes, cron jobs, privileged services, or SUID programs**.
