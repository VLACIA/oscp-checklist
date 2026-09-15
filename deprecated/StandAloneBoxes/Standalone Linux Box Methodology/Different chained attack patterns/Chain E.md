# Chain E — LFI/PHP Filter → Database Credentials → SQL File Write → RCE

**Attack path:** LFI → disclose PHP source → recover database credentials → database write primitive → web shell → local privilege escalation

## Brief explanation

1. **Confirm LFI:** A file parameter may allow local paths or PHP stream wrappers to be supplied to an unsafe include/read operation.
2. **Disclose PHP source:** The `php://filter` wrapper can base64-encode a PHP file before it is processed, allowing source such as `config.php` to be recovered and decoded.
3. **Extract database credentials:** Source code commonly reveals the database host, username, password, and schema name.
4. **Gain a file-write primitive:** See [[SQLi — UNION-Based]]. With sufficient database privileges, a MySQL injection or authenticated query may use `UNION ... INTO OUTFILE` to write a file. This requires the database account's `FILE` privilege, a permitted destination, and knowledge of the web root.
5. **Obtain and elevate a shell:** If the written file is a server-executable script, requesting it yields code execution under the web-service account; normal local enumeration is then used to find a separate privilege-escalation path.

**Why the chain works:** Source disclosure leaks a privileged database secret, while excessive database file permissions and an executable web directory convert database access into operating-system command execution.
