MSSQL is a **major OSCP attack path**: initial RCE via xp_cmdshell, lateral movement via linked servers, hash capture via UNC injection, privesc via impersonation.

What's happening here

**MSSQL is Microsoft's database server (port 1433) — but it can run operating-system commands.** If you get in as a privileged DB user (often `sa`), you can switch on a feature called `xp_cmdshell` that runs Windows commands _as the database service account_ — instant code execution.

**How you usually get in:** default/blank `sa` password, creds found in a web app's `web.config`, or a SQL injection that reaches the DB. Once inside, the steps below are a recipe: connect → enumerate → enable xp_cmdshell → shell. Even if you're not `sa`, the _impersonation_ trick (Step 4) can promote you.