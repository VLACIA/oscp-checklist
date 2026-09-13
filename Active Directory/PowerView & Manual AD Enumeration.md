# PowerView & Manual AD Enumeration

Use this after you have **valid domain credentials or a domain shell**. The goal is to build a domain map from the current user's perspective before choosing an attack.

> [!important] Repeat after every new identity
> A second low-privileged user is **not** necessarily equivalent to the first. Re-run users/groups/computers, local-admin access, sessions, SPNs, ACLs, shares, and BloodHound whenever you obtain a new identity.

## 1. Native Windows fallback — `net.exe`

These commands are useful when PowerView or RSAT is unavailable.

```cmd
:: Domain users
net user /domain

:: Inspect one user: group memberships, password/logon metadata, etc.
net user <USER> /domain

:: Domain groups
net group /domain

:: Members of a specific domain group
net group "<GROUP>" /domain
```

Look closely at names that suggest privileged or service accounts and enumerate interesting groups individually.

> [!note] Limitation
> `net.exe` is quick but incomplete. In particular, it can hide useful **nested-group** relationships. LDAP/PowerView enumeration is better when you need the full object graph.

## 2. LDAP / .NET fallback

AD queries are normally carried over **LDAP**. A reusable LDAP path is built from the PDC and the domain DN.

```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$LDAP = "LDAP://$PDC/$DN"
$LDAP
```

### Reusable LDAP search function

Useful when PowerView/RSAT is unavailable but PowerShell and .NET are present:

```powershell
function LDAPSearch {
    param([string]$LDAPQuery)

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName
    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DN")
    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)
    return $DirectorySearcher.FindAll()
}

# Users
LDAPSearch -LDAPQuery "(samAccountType=805306368)"

# Groups
LDAPSearch -LDAPQuery "(objectClass=group)"
```

The value `805306368` is the `samAccountType` used for normal user objects.

## 3. Import PowerView

```powershell
Import-Module .\PowerView.ps1
```

PowerView gives cleaner access to the same AD information and is usually faster for OSCP-style manual enumeration.

## 4. Domain, users, and groups

```powershell
# Domain / DC / PDC information
Get-NetDomain

# All users
Get-NetUser
Get-NetUser | select cn

# Useful account metadata
Get-NetUser | select cn,pwdlastset,lastlogon

# Groups
Get-NetGroup | select cn

# Members of one group
Get-NetGroup "Domain Admins" | select member
Get-NetGroup "<GROUP>" | select member
```

### What to look for

- privileged users/groups
- stale/dormant users (`pwdlastset`, `lastlogon`)
- service-looking usernames
- custom department/admin groups
- **nested groups**: group A may contain group B, indirectly granting members of B the permissions of A

Do not assume that a user belongs only to the groups shown by a quick `net group` check. Follow nested memberships until the chain ends.

## 5. Computers and operating systems

```powershell
# All computer objects
Get-NetComputer

# Useful compact view
Get-NetComputer | select operatingsystem,dnshostname

# Include OS version/build
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
```

Use this early to identify:

- the DC
- file/web/application servers
- older Windows hosts
- machines worth prioritizing for follow-up enumeration

## 6. Find where the current identity is local admin

```powershell
Find-LocalAdminAccess
```

This sprays domain computers and tests whether the current identity can obtain administrative access to the Service Control Manager.

**High-value relationship:**

```text
current user → AdminTo → computer
```

A machine where you are local admin becomes especially interesting if a privileged domain user has a session there.

## 7. Logged-on users / sessions

### PowerView

```powershell
Get-NetSession -ComputerName <HOST>
Get-NetSession -ComputerName <HOST> -Verbose
```

If there is no output, use `-Verbose`. Modern Windows permissions may cause `Access is denied`, so **no output does not automatically mean no sessions**.

### PsLoggedOn fallback

```cmd
PsLoggedon.exe \\<HOST>
```

`PsLoggedOn` can identify users through local logon information and resource-share sessions. It relies on the **Remote Registry** service for part of its enumeration, so results can be incomplete if that service is unavailable.

### Attack-path logic

```text
I am local admin on CLIENT74
        +
privileged user is logged on to CLIENT74
        ↓
potential credential / token theft path
```

Document these relationships even if you do not exploit them immediately.

## 8. SPN / service-account enumeration

Service Principal Names (SPNs) associate services with accounts. Enumerating them can reveal **service accounts, server names, service types, and sometimes ports** without a broad scan.

```cmd
:: SPNs registered to a specific account
setspn -L <USER>
```

```powershell
# All user accounts with SPNs
Get-NetUser -SPN | select samaccountname,serviceprincipalname
```

Resolve interesting hosts if necessary:

```cmd
nslookup <HOSTNAME>
```

If a normal domain user account has an SPN, investigate it as a Kerberoasting target:

→ [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]

## 9. Enumerate AD object ACLs

An AD object's permissions are represented by **ACEs** inside an **ACL**. For attack-path hunting, focus on who controls whom.

Interesting rights from Chapter 21:

- `GenericAll` — full control
- `GenericWrite` — modify selected attributes
- `WriteOwner` — change object owner
- `WriteDACL` — modify the object's ACL
- `AllExtendedRights` — includes sensitive extended operations such as password reset
- `ForceChangePassword` — change/reset the target password
- `Self` / self-membership — may allow adding yourself to a group

```powershell
# All ACEs applied to an object
Get-ObjectAcl -Identity <USER_OR_GROUP>

# Convert unreadable SID to DOMAIN\name
Convert-SidToName <SID>

# Example: find principals with GenericAll over a target group
Get-ObjectAcl -Identity "<GROUP>" |
    ? {$_.ActiveDirectoryRights -eq "GenericAll"} |
    select SecurityIdentifier,ActiveDirectoryRights
```

The most useful fields are usually:

- `ActiveDirectoryRights`
- `SecurityIdentifier` → convert with `Convert-SidToName`

When you find an exploitable relationship, continue here:

→ [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/ACL Abuse — Common BloodHound Paths|ACL Abuse — Common BloodHound Paths]]

## 10. Domain shares and SYSVOL

```powershell
# All discovered domain shares
Find-DomainShare

# Only shares the current user can access
Find-DomainShare -CheckShareAccess
```

Investigate **every readable non-default share**, not only `SYSVOL`:

- scripts
- configuration files
- backups
- onboarding documents/emails
- credentials and old passwords
- administrative documentation

### SYSVOL

Every domain user normally has read access to SYSVOL.

```powershell
ls \\<DC>\SYSVOL\<DOMAIN>\
ls \\<DC>\SYSVOL\<DOMAIN>\Policies\
```

Look for old policy files and GPP XML containing `cpassword` values.

→ [[Active Directory/Attack-Phases/Phase-2-3/GPP-SYSVOL|GPP / SYSVOL]]

## 11. Then automate and graph it

Manual enumeration teaches you what the relationships mean; SharpHound/BloodHound scales the same ideas and often exposes paths you missed visually.

→ [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]]

## Quick OSCP sequence

```text
1. whoami / current domain identity
2. users + groups + nested memberships
3. computers + OS
4. Find-LocalAdminAccess
5. logged-on sessions
6. SPNs → Kerberoasting candidates
7. ACLs → dangerous rights
8. shares + SYSVOL
9. SharpHound / BloodHound
10. mark owned → shortest path
11. gain new identity → repeat from step 2
```
