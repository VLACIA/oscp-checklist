**MSSQL Found (port 1433)**
        │
        │ Try creds:
        │ sa:(blank) / sa:sa / sa:password / domain_user:known_pass
        │
    ┌───┴──────────┐
 Sysadmin?      Not sysadmin?
    │               │
    │         Check IMPERSONATE rights
    │         → EXECUTE AS LOGIN='sa'
    │               │
    └───────────────┘
        │
 Enable xp_cmdshell → RCE on current server
        │
 Check linked servers (sp_linkedservers)
        │
    ┌───┴─────────────────────────────┐
 No links                       Links found
    │                                │
    │                        xp_cmdshell on linked
    │                        → Shell on MS02/DC
    │
 UNC injection → Responder → crack NetNTLMv2