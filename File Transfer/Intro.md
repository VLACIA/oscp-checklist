**After getting a shell, you'll always need to move files** — pushing your enumeration scripts (linpeas, winPEAS) _to_ the target, and pulling loot (hashes, configs) _back_ to Kali. The target rarely has internet, so you host the file on your own machine and have the target fetch it.

**The standard pattern:** on Kali, start a quick web server in the folder with your file — `python3 -m http.server 80` — then on the target use whatever download tool exists (`wget`, `curl`, `certutil`, PowerShell). The commands below cover every OS combination. Pick the one that matches your shell.

