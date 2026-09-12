
How to think about a box

**A "standalone box" is one machine to fully own — get a shell, then escalate to root.** Most Linux boxes are cracked through a web app on port 80/443, so that's usually where you start. The web attacks below are the common ways in: find the flaw, get code execution, land a shell.

**The loop never changes:** enumerate the web app → spot a weakness (file upload, LFI, SQLi, known CVE) → turn it into a _reverse shell_ → then run the PrivEsc checklist further down to become root.