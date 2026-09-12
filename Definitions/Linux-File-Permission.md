tag:#basic

On Linux, every file says _who_ can **r**ead, **w**rite, and e**x**ecute it — split into three groups: the **owner**, the **group**, and **everyone else**. Misconfigured permissions (e.g. a root-owned script _you_ can write to) are a classic way to escalate to root.

you can view permission like

```
ls -l
```

example:
-rwxrwxr-x 1 mkarami mkarami 5795 Sep 11 09:38 universal.ovpn
- `-` = regular file
- Owner: `rwx` = read, write, execute
- Group: `rwx` = read, write, execute
- Others: `r-x` = read and execute, but cannot write

The permission is equivalent to:

```
775
```

because `rwx = 7`, `rwx = 7`, and `r-x = 5`.