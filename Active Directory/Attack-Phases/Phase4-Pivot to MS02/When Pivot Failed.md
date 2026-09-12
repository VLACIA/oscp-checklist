```
### PTH to MS02 failed?

Try in this order:

1. **Kerberoast from MS01 context** → crack → try new creds on MS02
2. **MSSQL on MS01 linked to MS02** → `xp_cmdshell` on MS02 (Section 6)
3. **PS history on MS01** → `C:\Users\*\AppData\...\ConsoleHost_history.txt`
4. **web.config / app.config on MS01** → DB creds → reuse on MS02
5. **BloodHound** → MS01 machine account `GenericAll` on MS02?
6. **Unconstrained delegation on MS01** → coerce DC → TGT → DCSync
7. **ADCS ESC1/ESC8** → cert as MS02 admin → authenticate
8. **Password spray** with ALL found creds/hashes on MS02 services
```