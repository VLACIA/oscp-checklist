```
SELECT @@version;
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');    -- 1 = sysadmin!
SELECT name FROM master.dbo.sysdatabases;

-- Check impersonation rights:
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Linked servers:
EXEC sp_linkedservers;
SELECT srvname, isremote FROM master..sysservers;
```