# Phase 1 — Active Directory Enumeration

The goal is to build a **relationship map**, not just collect usernames. Chapter 21's core loop is:

```text
valid domain identity
    ↓
users / groups / computers
    ↓
local-admin access
    ↓
logged-on sessions
    ↓
SPNs / service accounts
    ↓
ACL relationships
    ↓
shares / SYSVOL
    ↓
SharpHound / BloodHound
    ↓
mark owned → identify attack path
    ↓
new identity / foothold
    └──────────────→ repeat enumeration
```

Detailed Windows commands: [[Active Directory/PowerView & Manual AD Enumeration|PowerView & Manual AD Enumeration]]  
Graph collection/analysis: [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]]  
End-to-end Chapter 24 workflow: [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces — End-to-End OSCP Attack Chain]]  

---

## 0. Establish your current identity and domain context

PowerShell/domain context:

```powershell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
([adsi]'').distinguishedName
```

Record the **current domain identity**, domain/DC context, and what access that identity has.

> [!important]
> If you gain **any new domain identity**, do not assume it has the same permissions as the old one. Start this phase again from that user's perspective.

## 1. Users, groups, and nested memberships

### Native Windows

```cmd
net user /domain
net user <USER> /domain
net group /domain
net group "<GROUP>" /domain
```

### PowerView

```powershell
Import-Module .\PowerView.ps1

Get-NetDomain
Get-NetUser | select cn
Get-NetUser | select cn,pwdlastset,lastlogon
Get-NetGroup | select cn
Get-NetGroup "<GROUP>" | select member
```

Look for:

- Domain Admins / Enterprise Admins and other privileged/custom groups
- admin/service-looking usernames
- stale/dormant users
- **nested groups** and indirect membership

`net.exe` is fast but can miss group-within-group relationships, so use LDAP/PowerView when you need the full picture.

## 2. Enumerate computers and operating systems

```powershell
Get-NetComputer | select operatingsystem,dnshostname
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
```

Identify DCs, file servers, web/app servers, and older workstations.

From Kali, keep doing normal service enumeration against reachable hosts/DCs as appropriate.

## 3. Where are you local administrator?

```powershell
Find-LocalAdminAccess
```

This is a high-value relationship because local admin on a machine may let you access credentials/tokens belonging to other domain users who use that machine.

## 4. Who is logged on where?

```powershell
Get-NetSession -ComputerName <HOST>
Get-NetSession -ComputerName <HOST> -Verbose
```

If PowerView session enumeration is blocked or misleading, try:

```cmd
PsLoggedon.exe \\<HOST>
```

Modern Windows permissions can restrict `NetSessionEnum`; `PsLoggedOn` also depends partly on Remote Registry. Treat results as evidence, not absolute truth.

**Very high-value finding:**

```text
you are local admin on HOST
+
privileged user has a session on HOST
=
potential credential/token theft path
```

## 5. Enumerate SPNs / service accounts

```cmd
setspn -L <USER>
```

```powershell
Get-NetUser -SPN | select samaccountname,serviceprincipalname
```

SPNs can reveal service accounts, hosts, service types, and ports. If you find a user-backed SPN:

→ [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]

From Kali you can also identify/request roastable accounts with Impacket:

```bash
GetUserSPNs.py corp.local/<USER>:'<PASSWORD>' -dc-ip <DC_IP>
```

## 6. Enumerate dangerous ACL relationships

PowerView:

```powershell
Get-ObjectAcl -Identity <OBJECT>
Convert-SidToName <SID>

Get-ObjectAcl -Identity "<GROUP>" |
    ? {$_.ActiveDirectoryRights -eq "GenericAll"} |
    select SecurityIdentifier,ActiveDirectoryRights
```

Prioritize:

```text
GenericAll
GenericWrite
WriteOwner
WriteDACL
AllExtendedRights
ForceChangePassword
Self / Self-Membership
```

Then map the relationship to an abuse technique:

→ [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/ACL Abuse — Common BloodHound Paths|ACL Abuse — Common BloodHound Paths]]

## 7. Enumerate shares and SYSVOL

PowerView:

```powershell
Find-DomainShare
Find-DomainShare -CheckShareAccess
```

Inspect all readable shares for:

```text
configs · scripts · backups · documentation · onboarding files · passwords · keys
```

SYSVOL is especially important:

```powershell
ls \\<DC>\SYSVOL\<DOMAIN>\
ls \\<DC>\SYSVOL\<DOMAIN>\Policies\
```

If you find GPP XML / `cpassword` artifacts:

→ [[Active Directory/Attack-Phases/Phase-2-3/GPP-SYSVOL|GPP / SYSVOL]]


## 8. Internal foothold situational awareness — do this before chasing AD attacks

Chapter 24 starts the internal phase by understanding the **local workstation and network context**, then uses that information to decide what to enumerate next.

Immediately capture:

```powershell
whoami
hostname
ipconfig
systeminfo
```

Do not trust automated enumeration blindly. In the Chapter 24 walkthrough, winPEAS reported the workstation as Windows 10 while `systeminfo` showed the correct Windows 11 version.

From network/DNS information, record:

```text
current IPv4 + subnet
Default Gateway
DNS server
cached/known hostnames
likely DC
other internal servers
```

Resolve interesting names:

```powershell
nslookup <HOSTNAME>
```

### Identify dual-homed hosts

If the same hostname is known with:

```text
external IP + internal IP
```

mark it as **dual-homed**. That is a high-value clue for routing, relay targeting, or pivot design.

Keep a simple host file such as:

```text
<IP> - <FQDN> - <ROLE> - <NOTES>
```

> [!important]
> You do **not** need local admin on every workstation. If a host has no useful privesc vector, follow the identity/session/service path that advances the assessment objective.

### Groups and GPOs still matter

Chapter 24 skips group/GPO deep enumeration only because it provides no additional value in that simulated environment. In a real target, **do not skip them by default**; group nesting and GPO rights/configuration can create privilege-escalation paths.

## 9. SharpHound / BloodHound — graph the relationships

Do **not** treat BloodHound as a substitute for understanding manual enumeration. It is excellent for scaling and visualizing the same relationships, but collection can be noisy.

### Windows-side SharpHound

```powershell
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\<USER>\Desktop\ -OutputPrefix "corp audit"
```

### Kali-side collector

```bash
bloodhound-python -u <USER> -p '<PASSWORD>' \
    -d corp.local -ns <DC_IP> -c All --zip
```

Use Windows-side SharpHound when Kali cannot directly reach the DC but your foothold can.

## 10. Mark Owned and identify the next attack path

After importing the data:

- mark every compromised **user** as **Owned**
- mark every compromised **computer** as **Owned** when appropriate
- run **Find Shortest Paths to Domain Admins**
- run **Find Shortest Paths to Domain Admins from Owned Principals**
- inspect the exact edges (`AdminTo`, session, ACL, membership, delegation, etc.)
- use the edge's help/abuse information to understand the required next step

Useful additional queries:

```text
Find All Kerberoastable Users
Find AS-REP Roastable Users
Computers with Unconstrained Delegation
Find Principals with DCSync Rights
```

## 11. Kali quick recon — keep these in the loop

```bash
# Users / groups / shares / password policy
nxc smb <DC_IP> -u <USER> -p '<PASSWORD>' \
    --shares --users --groups --pass-pol

# Passwords / secrets left in AD description fields
ldapsearch -x -H ldap://<DC_IP> \
    -D "<USER>@corp.local" -w '<PASSWORD>' \
    -b "DC=corp,DC=local" \
    "(&(objectClass=user)(description=*))" sAMAccountName description
```

More LDAP attributes to inspect:

→ [[Active Directory/Attack-Phases/Phase-2-3/LDAP — Extended Field Enumeration|LDAP — Extended Field Enumeration]]

## 12. New identity = re-enumerate

Whenever you obtain:

- plaintext password
- NTLM hash
- Kerberos ticket
- new domain user
- new admin rights
- a shell on another machine

validate it, mark the principal as owned, and **repeat this phase**. This "rinse and repeat" step is one of the most important AD habits in Chapter 21.

```text
new identity → users/groups/computers → admin access → sessions → SPNs → ACLs → shares → BloodHound → next path
```
