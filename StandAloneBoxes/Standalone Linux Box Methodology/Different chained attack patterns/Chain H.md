# Chain H — SNMP Disclosure → Username → Password Spray → Remote Access

**Attack path:** weak SNMP community string → system/process information → username → controlled password spray → SSH/WinRM → privilege escalation

## Brief explanation

1. **Enumerate SNMP:** A default or disclosed read community string can let `snmpwalk` query system descriptions, installed software, running processes, and sometimes process arguments.
2. **Derive valid usernames:** Home paths, service ownership, process commands, and contact fields may disclose real account names.
3. **Perform a limited password spray:** Try a very small number of likely passwords across known accounts while respecting lockout policy; spraying differs from rapidly brute-forcing one account.
4. **Use the matching remote service:** Successful Linux credentials may permit SSH; successful Windows credentials may permit WinRM if the account is authorized.
5. **Escalate on the correct platform:** Check SUID/`sudo`/capabilities on Linux or token privileges such as `SeImpersonatePrivilege` on Windows.

**Why the chain works:** Information exposed through SNMP improves credential guessing, while weak passwords and a local privilege-escalation condition turn reconnaissance into full compromise.
