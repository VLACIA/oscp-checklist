### Post-Exploitation on MS01 — Local PrivEsc → Loot → Bridge to MS02

**⚠ This is the step most students skip.** Getting a low-priv shell on MS01 isn't the goal — it's the _starting point_. You need to escalate locally on MS01 first, then loot it properly, because the credentials/tickets that unlock MS02 are usually sitting on MS01, not found through more AD enumeration. Same mental model applies whether MS01 is domain-joined Windows, domain-joined Linux, or a standalone box in a multi-machine chain.

Why local privesc first

**A low-priv domain user's shell on MS01 usually can't read LSASS, LSA secrets, or other users' saved creds — you need local admin/SYSTEM on MS01 itself first.** Run the exact same "Standalone Windows PrivEsc" checklist from Section 4 (SeImpersonate → Potato, service misconfigs, unattend.xml, etc.) — being domain-joined changes nothing about _how_ you escalate locally, it only changes what you go looking for _once you're admin_.