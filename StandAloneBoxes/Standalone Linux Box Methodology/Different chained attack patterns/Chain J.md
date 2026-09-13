# Chain J — SQL Injection → Database-Assisted OS Command Execution

**Attack path:** SQL injection → database administrative capability → OS command execution as the database service account → local privilege escalation

## Brief explanation

1. **Confirm SQL injection:** User-controlled input in a login form or other parameter changes the structure of a backend SQL query because it is not safely parameterized.
2. **Identify the database and privileges:** The route to operating-system access depends on the DBMS and on the compromised database account's permissions.
3. **Reach OS execution:** On Microsoft SQL Server, a sufficiently privileged login may enable and invoke `xp_cmdshell`. On MySQL, a UDF-based route requires the ability to write a compatible library to the plugin directory and load it; file-write-to-webroot may be an alternative in some configurations.
4. **Operate as the service identity:** Commands initially run with the privileges of the database service account, not automatically as root or `SYSTEM`.
5. **Escalate separately:** On Windows, an assigned privilege such as `SeImpersonatePrivilege` may provide a path to `SYSTEM`; on Linux, enumerate `sudo`, SUID, capabilities, cron, and service misconfigurations.

**Why the chain works:** Unsafe query construction exposes a highly privileged database context, dangerous database features bridge into the operating system, and a separate local weakness enables final elevation.
