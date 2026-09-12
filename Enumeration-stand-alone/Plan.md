
**Enumeration = finding doors before you try to kick them in.** The goal here is to build a complete picture of the target: what ports are open, what software is running, and what versions. Each open port is a potential way in.

**The #1 student mistake:** running one quick `nmap` scan, seeing port 80, and tunnel-visioning on the website for 3 hours — while the real foothold was a forgotten anonymous FTP on port 21. _Scan everything (all 65,535 ports + UDP top 20) before you commit to any one path._

**Golden rule:** for every service you find, ask three questions — _What version is it? Are there known exploits (`searchsploit`)? Can I log in with default/blank/anonymous creds?


tag:#enumeration 