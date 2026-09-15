# Database enumeration (1433, 3306, 5432 TCP)

## Fingerprint first

```bash
nmap -sV -p1433,3306,5432 --script ms-sql-info,mysql-info,pgsql-brute <TARGET_IP> -oN databases.txt
```

Avoid password-brute scripts unless explicitly permitted. Record the engine, exact version, TLS requirement, authentication mode, instance/database names, and whether remote login is restricted.

## Connect with discovered credentials

```bash
# MySQL / MariaDB
mysql -h <TARGET_IP> -u <USER> -p

# Microsoft SQL Server
impacket-mssqlclient '<DOMAIN>/<USER>:<PASSWORD>@<TARGET_IP>' -windows-auth

# PostgreSQL
psql -h <TARGET_IP> -U <USER> -d <DATABASE>
```

## After authentication

- Enumerate current identity, roles, databases, schemas/tables, linked or foreign servers, readable files, and credential-bearing application data.
- Check privileges such as MySQL `FILE`, MSSQL server roles/impersonation and `xp_cmdshell` state, or PostgreSQL superuser and large-object/file capabilities.
- Prefer read-only queries. Enabling execution features or writing files changes target state; do so only when justified and document it.
- Test blank/default credentials only when allowed, then reuse credentials deliberately across exposed services.

## PEN-200 Chapter 10 — useful post-login queries

### MySQL / MariaDB

```sql
SELECT version();
SELECT system_user();
SHOW DATABASES;

-- Example: inspect an account hash in mysql.user
SELECT user, authentication_string FROM mysql.user WHERE user='<USER>';
```

Remember: the MySQL `root` account is a **database** administrative account; it is not automatically the Linux `root` user.

### MSSQL

```sql
SELECT @@version;
SELECT name FROM sys.databases;

-- Enumerate tables in a chosen database
SELECT * FROM <DATABASE>.information_schema.tables;

-- Read a table using database.schema.table
SELECT * FROM <DATABASE>.dbo.<TABLE>;
```

With `sqlcmd`, statements are normally terminated and then submitted with `GO` on its own line. Remote clients using TDS (such as `impacket-mssqlclient`) do not require `GO`.

For exploitation through a web parameter rather than direct DB credentials, see [[SQL Injection]].
