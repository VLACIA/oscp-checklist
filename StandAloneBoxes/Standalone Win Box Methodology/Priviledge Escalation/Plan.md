Mental model

**Same idea as Linux, different doors.** You're a normal Windows user hunting for a path to `SYSTEM` (Windows' equivalent of root). Run **winPEAS** first, then check these in order:

**1. Token privileges** — run `whoami /priv`. If you see _SeImpersonatePrivilege_, you almost certainly win with a "Potato" exploit. **2. Service misconfigs** — unquoted service paths or services you can modify let you run code as SYSTEM. **3. Stored credentials** — `unattend.xml`, registry autologon, saved PowerShell history, web.config. **4. Missing patches** — old kernels are vulnerable to known exploits.

**First thing to type every time:** `whoami /priv` and `whoami /groups`. They often hand you the answer in seconds.