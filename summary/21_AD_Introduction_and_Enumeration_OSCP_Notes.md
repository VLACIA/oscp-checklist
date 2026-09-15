---
title: "PEN-200 Chapter 21 - Active Directory Introduction and Enumeration"
aliases:
  - "PEN-200 Ch21 AD Enumeration"
  - "OSCP AD Enumeration"
tags:
  - oscp
  - pen-200
  - active-directory
  - enumeration
  - ldap
  - powerview
  - sharphound
  - bloodhound
chapter: 21
source: "PEN-200 - Active Directory Introduction and Enumeration"
status: study-note
---

# PEN-200 Chapter 21 — Active Directory Introduction and Enumeration

> [!summary]
> This chapter introduces the structure of Active Directory (AD), then builds an enumeration workflow from simple built-in Windows commands to LDAP queries with PowerShell/.NET, PowerView, session/permission/share enumeration, and finally SharpHound + BloodHound. The central OSCP lesson is that AD enumeration is **relationship discovery**: users, groups, computers, sessions, ACLs, service accounts, shares, and nested memberships combine into attack paths.

## Chapter map

- [[#21.1 Active Directory - Introduction]]
  - [[#21.1.1 Enumeration - Defining our Goals]]
- [[#21.2 Active Directory - Manual Enumeration]]
  - [[#21.2.1 Active Directory - Enumeration Using Legacy Windows Tools]]
  - [[#21.2.2 Enumerating Active Directory using PowerShell and .NET Classes]]
  - [[#21.2.3 Adding Search Functionality to our Script]]
  - [[#21.2.4 AD Enumeration with PowerView]]
- [[#21.3 Manual Enumeration - Expanding our Repertoire]]
  - [[#21.3.1 Enumerating Operating Systems]]
  - [[#21.3.2 Getting an Overview - Permissions and Logged on Users]]
  - [[#21.3.3 Enumeration Through Service Principal Names]]
  - [[#21.3.4 Enumerating Object Permissions]]
  - [[#21.3.5 Enumerating Domain Shares]]
- [[#21.4 Active Directory - Automated Enumeration]]
  - [[#21.4.1 Collecting Data with SharpHound]]
  - [[#21.4.2 Analysing Data using BloodHound]]
- [[#21.5 Wrapping Up]]
- [[#Attack chain connection]]
- [[#OSCP workflow distilled from the chapter]]
- [[#Mini cheat sheet]]

---

# 21.1 Active Directory - Introduction

Active Directory Domain Services (AD DS), usually just called **Active Directory**, is both a directory service and a management layer for a Windows enterprise environment. It stores and manages objects such as:

- **Users** — identities that can authenticate to domain-joined systems.
- **Computers** — workstations and servers joined to the domain.
- **Groups** — collections of users/computers/groups to which permissions can be assigned.
- **Organizational Units (OUs)** — containers used to organize AD objects, comparable conceptually to folders.
- **Attributes** — properties attached to objects, such as usernames, names, phone numbers, group membership, OS information, SPNs, etc.

AD depends heavily on **DNS**. A Domain Controller (DC) commonly also hosts DNS and is authoritative for the domain. Domain Controllers are central because they store AD objects and their attributes and participate in authentication/authorization.

Important privilege concepts:

- **Domain Admins** — extremely privileged within one domain. Compromising a Domain Admin generally gives control of that domain.
- **Enterprise Admins** — forest-wide high privilege; members have extensive control across domains in the forest and administrative rights on DCs.
- A single AD deployment may contain multiple domains arranged into a **domain tree**, and multiple trees can form a **forest**.

The protocol that matters throughout this chapter is **LDAP — Lightweight Directory Access Protocol**. LDAP is the primary channel used to query directory information such as users, groups, computers, attributes, and relationships.

> [!important]
> On an OSCP-style AD target, do not think only in terms of “Which host can I exploit?” Think in terms of **objects and relationships**: who is in which group, who is admin where, which user has a session on which host, which ACE grants control over which object, and what credential-bearing files or shares are reachable.

---

## 21.1.1 Enumeration - Defining our Goals

The chapter assumes an **assumed-breach** position:

- Domain: `corp.com`
- Compromised low-privileged domain user: `stephanie`
- Initial workstation: Windows 11 `CLIENT75`
- `stephanie` can RDP to the system but is **not a local administrator** there.

The goal is to enumerate the domain and identify ways to reach the highest useful privilege, ultimately Domain Administrator in the training scenario.

The most important methodology point is **repeat enumeration after every pivot or new credential**. A newly compromised account may appear to be another low-privileged user, but AD permissions are often assigned individually or according to job role. That new identity may therefore expose new shares, local-admin access, ACL permissions, sessions, or group memberships.

### OSCP reminder

> [!tip]
> **Rinse and repeat:** every time you gain a user, computer, token, password, or new execution context, rerun the relevant AD enumeration. Your view of the domain changes with your identity and location.

---

# 21.2 Active Directory - Manual Enumeration

The chapter starts with tools already present in Windows, then progresses to PowerShell/.NET and LDAP. This gives both a practical workflow and an understanding of what automated tools are doing underneath.

---

## 21.2.1 Active Directory - Enumeration Using Legacy Windows Tools

### Connect to the compromised domain workstation with `xfreerdp`

```bash
xfreerdp /u:stephanie /d:corp.com /v:192.168.50.75
```

**Tool:** `xfreerdp` — Linux RDP client.

**Arguments:**

- `/u:stephanie` — username.
- `/d:corp.com` — Windows domain.
- `/v:192.168.50.75` — target RDP server.
- Password is entered interactively in the chapter.

**When/why:** use when you have valid Windows credentials and RDP is available. It provides an interactive user context inside the domain, from which many native enumeration commands can be executed.

### Enumerate all domain users with `net.exe`

```cmd
net user /domain
```

**Tool:** `net.exe` — built-in Windows administrative/network command utility.

**Arguments:**

- `user` — user-account subcommand.
- `/domain` — query the domain instead of the local SAM database.

**Why:** quickly obtain usernames. Naming patterns such as `admin`, `svc`, `sql`, `backup`, etc. can immediately identify accounts worth prioritizing.

### Inspect a specific domain user

```cmd
net user jeffadmin /domain
```

This returns account metadata including activity, password timing, local/global group memberships, and other attributes available through `net`.

In the chapter, `jeffadmin` is revealed as a member of **Domain Admins**, making it a high-value credential target.

### Enumerate domain groups

```cmd
net group /domain
```

**Arguments:**

- `group` — domain/global group operations.
- `/domain` — query the domain controller.

**Why:** identify built-in and custom groups. Custom departmental or administrative groups are especially interesting because their permissions often reflect real organizational roles.

### Enumerate members of a group

```powershell
net group "Sales Department" /domain
```

The chapter finds `pete` and `stephanie` as user members of the group.

> [!warning]
> `net group` is useful but incomplete for deeper AD mapping. In this chapter it fails to reveal a **nested group** that LDAP enumeration later finds. Use it for fast triage, not as your only source of truth.

---

## 21.2.2 Enumerating Active Directory using PowerShell and .NET Classes

### Why not simply use `Get-ADUser`?

`Get-ADUser` is part of the **ActiveDirectory PowerShell module / RSAT**. RSAT is commonly present on DCs or admin workstations but is not normally available on ordinary clients, and installing it can require administrative rights. The chapter therefore builds LDAP queries with built-in .NET capabilities instead.

### LDAP fundamentals

General LDAP ADsPath format:

```text
LDAP://HostName[:PortNumber][/DistinguishedName]
```

Components:

- `HostName` — DC hostname, IP, or domain name.
- `PortNumber` — optional; default depends on LDAP/LDAPS configuration.
- `DistinguishedName (DN)` — unique LDAP path to an AD object or search root.

Example DN:

```text
CN=Stephanie,CN=Users,DC=corp,DC=com
```

Interpretation from right to left:

- `DC=corp,DC=com` — domain components for `corp.com`.
- `CN=Users` — parent container.
- `CN=Stephanie` — object common name.

The chapter chooses the **PDC role owner** as the DC to query so the script points at the domain controller expected to hold the most current information.

### Get the current domain object

```powershell
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

**Components:**

- `System.DirectoryServices.ActiveDirectory` — .NET namespace for AD-related classes.
- `Domain` — class representing an AD domain.
- `::GetCurrentDomain()` — static method returning the domain associated with the current user context.

Useful returned properties include `PdcRoleOwner`, `DomainControllers`, `Forest`, and `Name`.

### Store the domain object

```powershell
# Store the domain object in the $domainObj variable
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Print the variable
$domainObj
```

### Bypass the PowerShell execution policy for the current shell

```powershell
powershell -ep bypass
```

**Arguments:**

- `-ep` — short form of `-ExecutionPolicy`.
- `bypass` — do not block script execution in that launched PowerShell process.

**When/why:** when a local execution policy prevents `.ps1` scripts from running. This is not privilege escalation; it changes PowerShell policy behavior for the launched process.

### Run the enumeration script

```powershell
.\enumeration.ps1
```

### Extract the PDC hostname

```powershell
# Store the domain object in the $domainObj variable
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Store the PdcRoleOwner name to the $PDC variable
$PDC = $domainObj.PdcRoleOwner.Name

# Print the $PDC variable
$PDC
```

This resolves to `DC1.corp.com` in the lab.

### Obtain the domain Distinguished Name with ADSI

```powershell
([adsi]'').distinguishedName
```

**Tool/concept:** `[adsi]` is a PowerShell type accelerator for **Active Directory Service Interfaces (ADSI)**. An empty ADSI path starts at the current domain context, and `.distinguishedName` retrieves the base DN.

Lab result:

```text
DC=corp,DC=com
```

### Store the DN

```powershell
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = $domainObj.PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$DN
```

### Build a reusable LDAP path

```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$LDAP = "LDAP://$PDC/$DN"
$LDAP
```

Lab output:

```text
LDAP://DC1.corp.com/DC=corp,DC=com
```

**Why this matters:** the script dynamically discovers the current domain’s PDC and DN rather than hard-coding one environment. That makes the enumeration logic portable.

---

## 21.2.3 Adding Search Functionality to our Script

The chapter uses two classes from `System.DirectoryServices`:

- **DirectoryEntry** — represents/binds to an object or location in the AD hierarchy.
- **DirectorySearcher** — performs LDAP-backed searches from a specified `SearchRoot`.

### Search the whole domain

```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$LDAP = "LDAP://$PDC/$DN"

$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.FindAll()
```

**Key pieces:**

- `New-Object System.DirectoryServices.DirectoryEntry($LDAP)` — bind to the LDAP search root.
- `New-Object ... DirectorySearcher($direntry)` — construct a searcher using that root.
- `.FindAll()` — return all matching entries.

With no filter, this returns a huge set of AD objects.

### Filter for normal user objects by `samAccountType`

```powershell
$PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$LDAP = "LDAP://$PDC/$DN"

$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="samAccountType=805306368"
$dirsearcher.FindAll()
```

`805306368` is decimal `0x30000000`, used here to select normal user objects.

### Print all properties for each returned object

```powershell
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = $domainObj.PdcRoleOwner.Name
$DN = ([adsi]'').distinguishedName
$LDAP = "LDAP://$PDC/$DN"

$direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="samAccountType=805306368"
$result = $dirsearcher.FindAll()

Foreach($obj in $result)
{
    Foreach($prop in $obj.Properties)
    {
        $prop
    }
    Write-Host "-------------------------------"
}
```

**Why:** AD objects contain many potentially useful attributes. Looking at all properties helps discover group membership, timestamps, SPNs, flags, SIDs, and other attack-relevant metadata.

The chapter highlights attributes from `jeffadmin`, especially:

- `memberof` → Domain Admins and Administrators.
- `samaccountname` → username.
- `distinguishedname` → LDAP identity/path.
- `admincount` → can be an indicator associated with protected/admin objects.
- `pwdlastset`, `lastlogon`, `lastlogontimestamp` → account activity clues.
- `useraccountcontrol` → account configuration flags.

### Filter one user and show only `memberof`

```powershell
$dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
$dirsearcher.filter="name=jeffadmin"
$result = $dirsearcher.FindAll()

Foreach($obj in $result)
{
    Foreach($prop in $obj.Properties)
    {
        $prop.memberof
    }
    Write-Host "-------------------------------"
}
```

This confirms `jeffadmin` belongs to `Domain Admins`.

### Turn the LDAP logic into a reusable function

```powershell
function LDAPSearch {
    param (
        [string]$LDAPQuery
    )

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DistinguishedName = ([adsi]'').distinguishedName
    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")
    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)

    return $DirectorySearcher.FindAll()
}
```

**Argument:**

- `$LDAPQuery` — caller-supplied LDAP filter string.

**Why:** this eliminates repeatedly editing the script for every query.

### Import the function

```powershell
Import-Module .\function.ps1
```

`Import-Module` loads the script/function definitions into the current PowerShell session.

### Enumerate users with the function

```powershell
LDAPSearch -LDAPQuery "(samAccountType=805306368)"
```

### Enumerate groups by object class

```powershell
LDAPSearch -LDAPQuery "(objectclass=group)"
```

This finds more than `net group` because LDAP can return **Domain Local groups and group objects**, not merely the subset shown by the legacy command.

### Enumerate group CNs and members

```powershell
foreach ($group in $(LDAPSearch -LDAPQuery "(objectCategory=group)")) {
    $group.properties | select {$_.cn}, {$_.member}
}
```

**PowerShell pieces:**

- `$(...)` — subexpression, evaluates the LDAP query first.
- `foreach` — iterate through each returned group object.
- `.properties` — access AD properties returned by `DirectorySearcher`.
- `select {$_.cn}, {$_.member}` — display only CN and membership data.

### Query the Sales Department specifically

```powershell
$sales = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Sales Department))"
$sales.properties.member
```

LDAP filter explanation:

- `&` — logical AND.
- `(objectCategory=group)` — object must be a group.
- `(cn=Sales Department)` — group CN must match exactly.

This exposes a **nested group** that `net group` missed: `Development Department` is itself a member of `Sales Department`.

### Continue following nested groups

```powershell
$group = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Development Department*))"
$group.properties.member
```

`*` is a wildcard after the CN value.

Then:

```powershell
$group = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Management Department*))"
$group.properties.member
```

The chain discovered is conceptually:

```text
jen
  -> member of Management Department
  -> Management Department is member of Development Department
  -> Development Department is member of Sales Department
```

This matters because **nested group membership can produce indirect privileges**.

> [!important]
> Never stop at direct membership. In AD, group nesting can make an apparently low-privileged user inherit permissions through several levels.

---

## 21.2.4 AD Enumeration with PowerView

**PowerView** is a PowerShell AD reconnaissance framework. It wraps much of the manual LDAP/.NET logic into reusable commands and exposes many attack-relevant relationships.

### Import PowerView

```powershell
Import-Module .\PowerView.ps1
```

### Get basic domain information

```powershell
Get-NetDomain
```

Useful for domain name, forest, DCs, PDC role owner, and related metadata.

### Enumerate all domain users

```powershell
Get-NetUser
```

PowerView returns many user attributes by default.

### Get a clean username list

```powershell
Get-NetUser | select cn
```

- `|` pipes objects to the next command.
- `select cn` displays only the `cn` property.

### Look for stale/dormant or potentially weak accounts

```powershell
Get-NetUser | select cn,pwdlastset,lastlogon
```

**Why:** old `pwdlastset` or old `lastlogon` can identify dormant accounts, potentially outdated passwords, and accounts that may be less monitored.

### Enumerate groups

```powershell
Get-NetGroup | select cn
```

### Enumerate a specific group’s members

```powershell
Get-NetGroup "Sales Department" | select member
```

PowerView reveals nested group membership just like the custom LDAP query.

---

# 21.3 Manual Enumeration - Expanding our Repertoire

The focus now changes from merely listing objects to building a **domain map**. The most useful information is often the relationship between objects:

- User → group
- Group → nested group
- User → local admin on computer
- User → active session on computer
- User/group → ACL control over object
- Service account → SPN/service/host
- User → accessible share → credential/configuration data

---

## 21.3.1 Enumerating Operating Systems

### Enumerate computer objects

```powershell
Get-NetComputer
```

This returns computer-account attributes such as hostname, OS, OS version, SPNs, account flags, timestamps, and more.

### Show only OS and DNS hostname

```powershell
Get-NetComputer | select operatingsystem,dnshostname
```

The lab contains six systems, including DC, web/file servers, Windows 11 workstations, and a Windows 10 client.

**Why:** AD can provide a fast system inventory without scanning every host. Older operating systems or specially named servers (`web`, `files`, `sql`, etc.) become priority targets for later enumeration.

### Show OS version/build as well

```powershell
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
```

**Why:** build/version details help judge whether older enumeration techniques or vulnerabilities may apply.

---

## 21.3.2 Getting an Overview - Permissions and Logged on Users

This subsection is about **chained compromise**. You do not necessarily need one jump from low privilege to Domain Admin. A realistic path may be:

```text
low-priv user -> admin on workstation -> credentials/session of stronger user -> privileged server -> Domain Admin
```

### Find computers where the current user is a local administrator

```powershell
Find-LocalAdminAccess
```

**PowerView behavior:** tries to connect to the target Service Control Manager (SCM) and open it with `SC_MANAGER_ALL_ACCESS`. Success indicates administrative access.

**Relevant underlying API:** `OpenServiceW` / Windows service-management functions.

**Optional parameters mentioned:**

- `-ComputerName` — target specific system(s).
- credentials parameter(s) can be supplied if not using current context.

The chapter runs it without parameters and finds:

```text
client74.corp.com
```

So `stephanie` is local admin on `CLIENT74`.

### Enumerate sessions with PowerView

```powershell
Get-NetSession -ComputerName files04
Get-NetSession -ComputerName web04
```

Underlying APIs discussed:

- `NetWkstaUserEnum` — logged-on user enumeration; commonly requires admin rights.
- `NetSessionEnum` — network session enumeration; permissions differ by query level and modern Windows configuration.

### Add verbose output when no result appears

```powershell
Get-NetSession -ComputerName files04 -Verbose
Get-NetSession -ComputerName web04 -Verbose
```

The chapter receives `Access is denied`, demonstrating that **no output does not necessarily mean no sessions**. Always consider permissions and tool limitations.

### Test the host where local-admin access exists

```powershell
Get-NetSession -ComputerName client74
```

The returned session information is not fully reliable in this case, motivating deeper investigation.

### Why `NetSessionEnum` fails on modern systems

The chapter explains that access to session information is controlled by permissions associated with the registry path:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity
```

### Inspect registry ACLs with `Get-Acl`

```powershell
Get-Acl -Path HKLM:SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\ | fl
```

**Arguments/components:**

- `Get-Acl` — retrieves an object’s access-control/security descriptor.
- `-Path` — target registry path.
- `HKLM:` — PowerShell registry provider for `HKEY_LOCAL_MACHINE`.
- `| fl` — pipe to `Format-List` for readable output.

The default permissions on modern Windows prevent ordinary remote domain users from reading the required session information. This is why older techniques may fail even when the command syntax is correct.

> [!tip]
> Keep `Get-NetSession` in your toolkit. Real environments often contain older OS versions, relaxed ACLs, or non-default configurations.

### PsLoggedOn as an alternative

**PsLoggedOn** is part of Sysinternals/PSTools. It:

- inspects `HKEY_USERS` via Remote Registry to identify locally logged-on users by SID;
- also uses `NetSessionEnum` for resource-share sessions.

**Limitation:** it depends on the **Remote Registry** service for the registry-based portion. That service is not enabled by default on many modern workstations but may be enabled by administrators or available on server systems.

#### FILES04

```powershell
.\PsLoggedon.exe \\files04
```

The chapter discovers `CORP\jeff` logged on locally.

#### WEB04

```powershell
.\PsLoggedon.exe \\web04
```

No local user is found in the lab output.

#### CLIENT74

```powershell
.\PsLoggedon.exe \\client74
```

The key finding is:

```text
CORP\jeffadmin
```

logged on locally, while `stephanie` appears via a resource share.

This creates a highly valuable attack path:

```text
stephanie
  -> local admin on CLIENT74
  -> jeffadmin session exists on CLIENT74
  -> jeffadmin is Domain Admin
```

The chapter deliberately stops short of credential theft here and continues enumeration, illustrating disciplined methodology.

---

## 21.3.3 Enumeration Through Service Principal Names

### Service accounts and SPNs

Applications/services need a security context. Built-in service identities include:

- `LocalSystem`
- `LocalService`
- `NetworkService`

More complex applications may use **domain user service accounts**. When a domain-integrated service such as IIS, SQL Server, or Exchange is associated with an account, AD can register a **Service Principal Name (SPN)**. An SPN identifies a specific service instance and ties it to an AD account.

This is valuable because SPNs expose service/host information directly from AD and later become important for AD authentication attacks.

The chapter also distinguishes:

- **Managed Service Accounts (MSA)** — introduced in Server 2008 R2.
- **Group Managed Service Accounts (gMSA)** — introduced later to better support multi-server/redundant services.
- Some environments still use ordinary domain users as service accounts.

### List SPNs associated with a specific account

```cmd
setspn -L iis_service
```

**Tool:** `setspn.exe` — built-in Windows SPN management/query utility.

**Argument:**

- `-L` — list SPNs registered to the specified account.

Lab result includes:

```text
HTTP/web04.corp.com
HTTP/web04
HTTP/web04.corp.com:80
```

This identifies an HTTP service on `web04`, including TCP port 80.

### Enumerate all user accounts with SPNs via PowerView

```powershell
Get-NetUser -SPN | select samaccountname,serviceprincipalname
```

**Arguments:**

- `-SPN` — filter to users with registered SPNs.
- `select samaccountname,serviceprincipalname` — show the account and SPN values only.

The lab identifies `iis_service` and `krbtgt` as SPN-bearing accounts.

### Resolve the SPN hostname

```powershell
nslookup.exe web04.corp.com
```

This resolves the AD-referenced service hostname to an IP address, allowing direct follow-up enumeration of the service.

> [!important]
> SPN enumeration can reveal servers and ports **without a broad network port scan**. It is also a bridge from pure AD enumeration to later authentication attacks such as Kerberoasting.

---

## 21.3.4 Enumerating Object Permissions

### ACEs and ACLs

An AD object has an **Access Control List (ACL)** composed of **Access Control Entries (ACEs)**. Each ACE states that an identity is allowed or denied specific rights over the object.

The chapter emphasizes the following high-value AD rights:

| Right | Practical meaning |
|---|---|
| `GenericAll` | Full control over the object |
| `GenericWrite` | Modify certain writable attributes |
| `WriteOwner` | Change object ownership |
| `WriteDACL` | Modify the object’s ACL/ACEs |
| `AllExtendedRights` | Extended rights such as password reset/change operations |
| `ForceChangePassword` | Force/reset the object’s password |
| `Self` / Self-Membership | Can permit adding oneself to a group |

These rights are often more important than traditional “admin group” membership because a misconfigured ACL can create a direct privilege-escalation path.

### Enumerate ACLs on a user

```powershell
Get-ObjectAcl -Identity stephanie
```

**Tool:** PowerView `Get-ObjectAcl`.

**Argument:**

- `-Identity stephanie` — AD object whose ACL is being enumerated.

Important output fields:

- `ObjectSID` — SID of the target object.
- `ActiveDirectoryRights` — right granted/denied.
- `SecurityIdentifier` — SID of the principal to which the ACE applies.
- `AceQualifier`, `AceType`, `IsInherited` — additional ACE characteristics.

### Convert an object SID to a readable name

```powershell
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104
```

Result: `CORP\stephanie`.

### Convert the ACE principal SID

```powershell
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-553
```

This resolves to `CORP\RAS and IAS Servers` in the lab.

### Search one group for `GenericAll`

```powershell
Get-ObjectAcl -Identity "Management Department" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
```

PowerShell pieces:

- `?` — alias for `Where-Object`.
- `$_` — current pipeline object.
- `-eq "GenericAll"` — keep only ACEs whose AD rights equal `GenericAll`.
- `select SecurityIdentifier,ActiveDirectoryRights` — reduce output to the useful fields.

The chapter finds several `GenericAll` principals, including `stephanie`.

### Convert multiple SIDs at once

```powershell
"S-1-5-21-1987370270-658905905-1781884369-512","S-1-5-21-1987370270-658905905-1781884369-1104","S-1-5-32-548","S-1-5-18","S-1-5-21-1987370270-658905905-1781884369-519" | Convert-SidToName
```

The resolved identities include:

- `CORP\Domain Admins`
- `CORP\stephanie`
- `BUILTIN\Account Operators`
- `Local System`
- `CORP\Enterprise Admins`

The notable misconfiguration is that a regular domain user (`stephanie`) has `GenericAll` over the `Management Department` group.

### Abuse `GenericAll` by adding yourself to the group

```powershell
net group "Management Department" stephanie /add /domain
```

**Arguments:**

- group name — target domain group.
- `stephanie` — user to add.
- `/add` — add membership.
- `/domain` — apply operation to domain group, not local group.

### Verify membership

```powershell
Get-NetGroup "Management Department" | select member
```

### Clean up the change

```powershell
net group "Management Department" stephanie /del /domain
```

- `/del` — delete/remove group membership.

### Verify cleanup

```powershell
Get-NetGroup "Management Department" | select member
```

> [!important]
> ACL abuse is a core AD escalation concept. Even if the demonstrated group does not immediately grant higher domain privilege, the exact same discovery process can reveal rights over privileged users, groups, computers, GPOs, or other attack-critical objects.

---

## 21.3.5 Enumerating Domain Shares

Shares often expose:

- scripts;
- backups;
- configuration files;
- credentials;
- password-policy clues;
- deployment tooling;
- documents describing infrastructure.

### Enumerate shares across the domain

```powershell
Find-DomainShare
```

**Tool:** PowerView.

**Optional flag discussed:**

- `-CheckShareAccess` — restrict results to shares the current user can access.

The chapter deliberately omits the flag to obtain a broader target list for later investigation.

Interesting lab shares include:

- `SYSVOL`
- `NETLOGON`
- `backup`
- `docshare`
- `Tools`
- `Users`
- other standard administrative shares (`ADMIN$`, `C$`, `IPC$`).

### Enumerate SYSVOL

```powershell
ls \\dc1.corp.com\sysvol\corp.com\
```

`SYSVOL` is commonly readable by domain users and stores Group Policy data and scripts.

### Enumerate policy folders

```powershell
ls \\dc1.corp.com\sysvol\corp.com\Policies\
```

The lab finds an `oldpolicy` directory.

### Read a historical GPP XML file

```powershell
cat \\dc1.corp.com\sysvol\corp.com\Policies\oldpolicy\old-policy-backup.xml
```

The XML includes a Group Policy Preferences `cpassword` value:

```xml
cpassword="+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
```

Historically, GPP could store local-account passwords encrypted with AES. Because the decryption key became public, these old artifacts are recoverable if left on a share.

### Decrypt a GPP `cpassword` on Kali

```bash
gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
```

Lab plaintext:

```text
P@$$w0rd
```

**Tool:** `gpp-decrypt` — Kali utility for decrypting legacy GPP `cpassword` strings.

**When/why:** whenever you find `Groups.xml`, old GPP XML, or a `cpassword=` value in SYSVOL/backups.

### Inspect a custom file share

```powershell
ls \\FILES04\docshare
```

Then:

```powershell
ls \\FILES04\docshare\docs\do-not-share
```

### Read the discovered text file

```powershell
cat \\FILES04\docshare\docs\do-not-share\start-email.txt
```

The file contains an old onboarding email with a possible cleartext password:

```text
HenchmanPutridBonbon11
```

The chapter notes that the password may have been changed, but it still provides:

- a candidate password to test carefully;
- clues about organizational password patterns;
- material for targeted password guessing/wordlists.

> [!tip]
> During OSCP AD work, recursively inspect readable non-default shares and `SYSVOL`. “Old”, “backup”, “archive”, “do-not-share”, onboarding docs, scripts, deployment folders, and configuration files are high-value names.

---

# 21.4 Active Directory - Automated Enumeration

Manual enumeration teaches what the data means, but large domains become difficult to reason about manually. Automated tooling helps collect and visualize relationships.

The chapter briefly mentions **PingCastle** as another AD assessment/reporting tool, but focuses on **SharpHound + BloodHound**.

Important trade-off:

> [!warning]
> Automated collectors can create substantial network traffic and are easier for defenders to notice. Use manual and automated enumeration together, choosing the approach appropriate to scope and OPSEC requirements.

---

## 21.4.1 Collecting Data with SharpHound

**SharpHound** is BloodHound’s data collector. It uses LDAP and Windows APIs similar to the manual techniques already demonstrated, including session and Remote Registry-related enumeration.

### Import the SharpHound PowerShell script

```powershell
Import-Module .\Sharphound.ps1
```

### Inspect the collector command/help

```powershell
Get-Help Invoke-BloodHound
```

Important parameters shown by the help include:

- `-CollectionMethod <String[]>` — what data/relationships to collect.
- `-Domain <String>` — target domain.
- `-SearchForest` — search forest-wide.
- `-Stealth` — reduce some noisy collection behavior.
- `-LdapFilter <String>` — custom LDAP filter.
- `-DistinguishedName <String>` — constrain search to a DN.
- `-ComputerFile <String>` — supply target computers from a file.
- `-OutputDirectory <String>` — output directory.
- `-OutputPrefix <String>` — prefix generated filenames.
- `-CacheName <String>` / `-MemCache` / `-RebuildCache` — collector caching behavior.
- `-RandomFilenames` — randomize output filenames.
- `-ZipFilename <String>` / `-NoZip` / `-ZipPassword <String>` — archive behavior.
- `-LdapUsername` / `-LdapPassword` — alternate LDAP credentials.
- `-DomainController <String>` — explicitly select a DC.
- `-LdapPort <Int32>` — non-default LDAP port.
- `-SecureLdap` — use LDAPS.
- `-DisableCertVerification` — disable certificate verification for secure LDAP.
- `-DisableSigning` — LDAP signing-related behavior.
- `-SkipPortCheck`, `-PortCheckTimeout` — connectivity checking behavior.
- `-ExcludeDCs` — omit domain controllers from certain collection.
- `-Throttle`, `-Jitter`, `-Threads` — tune rate/concurrency/noise.
- `-SkipRegistryLoggedOn` — skip Remote Registry logged-on collection.
- `-CollectAllProperties` — collect additional LDAP properties.
- `-Loop`, `-LoopDuration`, `-LoopInterval` — repeat collection over time.
- `-StatusInterval`, `-Verbosity` — reporting/log detail.

The help text explains that the PowerShell function loads the compiled C# ingestor in memory via reflection.

### Collect all supported relationship data for the lab

```powershell
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\stephanie\Desktop\ -OutputPrefix "corp audit"
```

**Arguments:**

- `-CollectionMethod All` — run the broad set of collection methods used in the chapter (all except certain local group policy collection noted in the text).
- `-OutputDirectory ...` — save results to the user’s Desktop.
- `-OutputPrefix "corp audit"` — add a recognizable prefix.

The collection resolves methods including groups, local admins, sessions, logged-on users, trusts, ACLs, containers, RDP, object properties, DCOM, SPN targets, and PowerShell remoting-related relationships.

### Inspect SharpHound output files

```powershell
ls C:\Users\stephanie\Desktop\
```

The collector creates:

- a `.zip` containing JSON graph data for BloodHound;
- a `.bin` cache file used to speed repeated data collection.

The cache is not required for BloodHound analysis itself.

### Looping collection

SharpHound can run in a loop to capture changing state, especially user sessions. A one-time run is only a **snapshot**; a privileged user who logs on ten minutes later will not appear in that snapshot.

> [!tip]
> If scope/OPSEC allows it and session information is important, looped collection can expose transient relationships that a single snapshot misses.

---

## 21.4.2 Analysing Data using BloodHound

**BloodHound** models AD data as a graph:

- **nodes** — users, computers, groups, domains, etc.
- **edges** — relationships/rights such as membership, admin rights, sessions, ACL control.

**Neo4j** is the graph database used by the BloodHound version in this chapter.

### Start Neo4j

```bash
sudo neo4j start
```

The service exposes its web interface on:

```text
http://localhost:7474
```

The chapter uses the initial default credentials `neo4j` / `neo4j`, then requires setting a new password on first login.

### Start BloodHound

```bash
bloodhound
```

Authenticate BloodHound to the Neo4j database using the Neo4j username/password.

### Import SharpHound data

Transfer the SharpHound `.zip` from Windows to Kali, then use BloodHound’s **Upload Data** function or drag-and-drop the ZIP into the GUI.

### Database Info

The **More Info / Database Info** view summarizes collected data: user count, groups, sessions, ACL relationships, etc. Use **Refresh Database Stats** if data is still updating.

### Find all Domain Admins

Use BloodHound’s built-in analysis query:

```text
Find all Domain Admins
```

In the chapter, `jeffadmin` and the built-in Administrator account connect to the `Domain Admins` group through membership edges.

### Display node labels

In BloodHound settings, **Node Label Display → Always Display** makes graph interpretation easier.

### Find shortest paths to Domain Admins

Run:

```text
Find Shortest Paths to Domain Admins
```

The graph exposes attack-path relationships that may be difficult to notice manually.

The edge between `stephanie` and `CLIENT74` is labeled:

```text
AdminTo
```

Meaning `stephanie` has administrative control over `CLIENT74`.

BloodHound’s edge **Help** view can provide:

- what the relationship means;
- potential abuse method(s);
- OPSEC/detection notes;
- references.

The graph also shows that `jeffadmin` has a session on `CLIENT74`, suggesting that a local admin on that host may be able to target cached/active credentials or impersonation opportunities.

### Mark owned principals

For objects you actually control:

- Search for the user/computer.
- Right-click it.
- Use **Mark User as Owned** or **Mark Computer as Owned**.

The chapter marks:

- `stephanie` as owned.
- `CLIENT75` as owned/controlled for path analysis purposes.

Owned nodes display a skull icon.

### Find paths from what you own

Run:

```text
Shortest Paths to Domain Admins from Owned Principals
```

The resulting chapter path is:

```text
CLIENT75
  -> stephanie has a session / is controlled
  -> stephanie AdminTo CLIENT74
  -> jeffadmin has a session on CLIENT74
  -> jeffadmin MemberOf Domain Admins
  -> Domain Admin
```

This is the chapter’s core lesson: BloodHound turns isolated enumeration facts into an **actionable attack chain**.

> [!important]
> Mark every object you truly control as owned. A path may only become visible once BloodHound knows your actual footholds.

---

# 21.5 Wrapping Up

The chapter progresses through four levels of AD enumeration:

1. **Native/legacy Windows commands** — fast triage of users and groups.
2. **PowerShell + .NET + LDAP** — understand and build flexible raw directory queries.
3. **PowerView and specialized tools** — enumerate users, computers, sessions, local-admin rights, SPNs, ACLs, and shares efficiently.
4. **SharpHound + BloodHound** — collect relationships at scale and visualize attack paths.

The larger message is that enumeration is not a one-time phase. It must be repeated as new accounts and hosts are compromised, because each identity has a different view and different permissions.

The information discovered here is intentionally preparatory. Later AD modules use it to attack authentication mechanisms and move laterally.

---

# Tools and concepts reference

| Tool / concept | What it does | When it is useful |
|---|---|---|
| `xfreerdp` | RDP client from Linux | Interactive access with valid Windows credentials |
| `net user` | Lists/queries users | Fast username/account triage |
| `net group` | Lists/queries/modifies domain groups | Group discovery, membership checks, ACL-abuse actions |
| LDAP | Directory query protocol | Core protocol for AD object/attribute enumeration |
| ADSI | Windows interface layer to directory services | Easy PowerShell binding/query support without RSAT |
| `System.DirectoryServices.ActiveDirectory.Domain` | .NET AD domain class | Obtain domain/PDC information dynamically |
| `DirectoryEntry` | .NET wrapper for an AD directory object/search root | Bind to LDAP path |
| `DirectorySearcher` | .NET LDAP search class | Query AD objects and attributes |
| PowerView | PowerShell AD recon toolkit | Broad manual AD enumeration and relationship discovery |
| `Get-NetDomain` | PowerView domain metadata | Identify domain/forest/DC/PDC |
| `Get-NetUser` | PowerView user enumeration | Users, timestamps, attributes, SPNs |
| `Get-NetGroup` | PowerView group enumeration | Groups and nested memberships |
| `Get-NetComputer` | PowerView computer enumeration | Hostnames, OS, versions, SPNs |
| `Find-LocalAdminAccess` | Tests current user’s local-admin rights across hosts | Find immediate lateral movement targets |
| `Get-NetSession` | Session enumeration | Locate logged-on users when OS/permissions allow it |
| `Get-Acl` | Reads Windows ACLs | Diagnose local/registry permissions and tool failures |
| PsLoggedOn | Sysinternals logon enumeration | Alternative session/logged-on-user discovery |
| `setspn` | Query/manage SPNs | Find service accounts and mapped services |
| `nslookup` | DNS resolver | Convert SPN hostnames to IPs |
| `Get-ObjectAcl` | PowerView AD ACL enumeration | Find object-control escalation paths |
| `Convert-SidToName` | Resolve SID to readable principal | Interpret ACL owners/subjects |
| `Find-DomainShare` | Find SMB shares across domain systems | Locate credential/configuration/document sources |
| `gpp-decrypt` | Decrypt legacy GPP `cpassword` | Recover passwords from old Group Policy Preferences files |
| SharpHound | BloodHound data collector | Automated relationship collection at scale |
| BloodHound | AD graph/path analysis | Identify shortest privilege/lateral movement paths |
| Neo4j | Graph database used by this BloodHound version | Stores graph nodes/edges for analysis |
| PingCastle | AD security assessment/reporting tool mentioned in chapter | Automated AD posture review; not the focus of this module |

---

# Attack chain connection

The chapter fits into the larger process:

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
```

More precisely, Chapter 21 lives mainly in **AD Enumeration / path discovery after initial access**, but it feeds several later stages.

| Attack-chain phase | Chapter 21 connection |
|---|---|
| **Recon** | AD itself becomes an internal source of infrastructure recon: users, computers, OS versions, servers, SPNs, DNS names, and shares. |
| **Enumeration** | This is the chapter’s main phase: LDAP, PowerView, sessions, SPNs, ACLs, group nesting, shares, SharpHound, BloodHound. |
| **Initial Access** | The chapter assumes this has already happened through a compromised domain user (`stephanie`) and access to `CLIENT75`. |
| **Privilege Escalation** | `Find-LocalAdminAccess`, ACL enumeration, `GenericAll`, nested groups, and BloodHound paths expose escalation opportunities. |
| **Credentials** | Logged-on-user/session discovery identifies where valuable credentials may be cached. SYSVOL/GPP and file shares directly expose password material. |
| **Pivoting** | Local admin on another host (CLIENT74) is a pivot opportunity; every new host/user becomes a new enumeration vantage point. |
| **AD** | SPNs, ACLs, group membership, service accounts, domain/forest metadata, and shortest paths prepare you for Kerberos attacks and lateral movement. |
| **Proof** | Later, once the path is exploited, you validate impact with the required proof files/evidence. Enumeration tells you the likely route to get there. |

### Chapter attack path in one diagram

```text
[Initial access]
stephanie @ CLIENT75
        |
        |  Find-LocalAdminAccess / BloodHound AdminTo
        v
     CLIENT74
        |
        |  PsLoggedOn / SharpHound session data
        v
jeffadmin session present
        |
        |  jeffadmin ∈ Domain Admins
        v
Potential Domain Admin compromise
```

Parallel findings provide other branches:

```text
AD LDAP/PowerView
  ├─ users / groups / nested groups
  ├─ computer OS inventory
  ├─ SPN -> iis_service -> web04:80
  ├─ ACL -> stephanie GenericAll over Management Department
  └─ shares
       ├─ SYSVOL -> old GPP cpassword -> gpp-decrypt
       └─ FILES04 docshare -> possible cleartext password
```

---

# OSCP workflow distilled from the chapter

1. **Establish your current identity and domain context.** Know user, domain, host, and whether you are local admin.
2. **Do cheap enumeration first.** `net user /domain`, `net group /domain`, inspect obvious admin/service accounts.
3. **Identify the domain/DC/PDC and base DN.** Understand the LDAP search root.
4. **Enumerate users, groups, and nested groups.** Do not trust only direct membership.
5. **Enumerate computers and OS versions.** Identify DCs, servers, old clients, web/file/database targets.
6. **Find where your user is admin.** Local-admin relationships are immediate lateral movement opportunities.
7. **Find logged-on privileged users.** A high-value user session on a host you administer can be the key path.
8. **Enumerate SPNs.** Map service accounts to hosts/services and prepare for later Kerberos attacks.
9. **Enumerate ACLs.** Look for `GenericAll`, `GenericWrite`, `WriteDACL`, `WriteOwner`, password-reset rights, and self-membership.
10. **Enumerate shares and SYSVOL.** Search for passwords, backups, scripts, deployment configs, GPP artifacts, and documentation.
11. **Run SharpHound/BloodHound when appropriate.** Use it to correlate relationships and discover paths you missed manually.
12. **Mark controlled principals as owned.** Run shortest-path queries from what you actually control.
13. **Document every finding.** Users, passwords, hashes, ACL rights, sessions, shares, admins, services, and host roles.
14. **After every new compromise, repeat the loop from the new context.**

---

# Mini cheat sheet

> [!note]
> These are the commands/reminders from this chapter that are most useful to keep beside you while working an AD machine.

```text
# 1. Fast domain user list
net user /domain

# 2. Inspect one user
net user <user> /domain

# 3. Fast domain group list
net group /domain

# 4. Inspect group membership
net group "<group>" /domain

# 5. PowerView domain info
Get-NetDomain

# 6. Users + useful account-age fields
Get-NetUser | select cn,pwdlastset,lastlogon

# 7. Groups / nested membership
Get-NetGroup | select cn
Get-NetGroup "<group>" | select member

# 8. Computer/OS inventory
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion

# 9. Where am I local admin?
Find-LocalAdminAccess

# 10. Session checks
Get-NetSession -ComputerName <host> -Verbose
.\PsLoggedon.exe \\<host>

# 11. SPN/service-account enumeration
Get-NetUser -SPN | select samaccountname,serviceprincipalname
setspn -L <account>

# 12. Resolve a discovered hostname
nslookup.exe <host.domain>

# 13. ACL enumeration for a target AD object
Get-ObjectAcl -Identity "<user-or-group>"

# 14. Hunt GenericAll quickly
Get-ObjectAcl -Identity "<target>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights

# 15. Resolve ACL SID
Convert-SidToName <SID>

# 16. Enumerate domain shares
Find-DomainShare

# 17. Always inspect SYSVOL
ls \\<dc>\sysvol\<domain>\

# 18. Legacy GPP password artifact
gpp-decrypt "<cpassword>"

# 19. SharpHound collection
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory <dir> -OutputPrefix "<name>"

# 20. BloodHound workflow
sudo neo4j start
bloodhound
# Import ZIP -> mark owned principals -> run shortest paths to Domain Admins
```

## Reminders beside the cheat sheet

- **Re-enumerate after every new user/host.**
- **Nested groups matter.** Direct group listings can miss indirect privileges.
- **No session output may mean “access denied”, not “no session”. Use `-Verbose`.**
- **A local-admin edge + privileged user session is a high-value credential path.**
- **SPN = service-account/service mapping and a future Kerberos attack lead.**
- **ACLs can be privilege escalation even when group membership looks harmless.**
- **SYSVOL and non-default shares are credential hunting territory.**
- **BloodHound is strongest when you mark what you truly own and reason from those nodes.**

---

# One-page mental model

```text
AD = objects + attributes + permissions + relationships

Objects:
  users | groups | computers | OUs | services

Relationships to hunt:
  MemberOf
  AdminTo
  HasSession
  ACL control (GenericAll/WriteDACL/etc.)
  SPN -> service host
  Share access -> files/credentials

Manual discovery:
  net.exe
  LDAP / ADSI / .NET
  PowerView
  PsLoggedOn
  setspn

Automated correlation:
  SharpHound -> ZIP/JSON -> BloodHound/Neo4j -> shortest attack path

Core OSCP loop:
  enumerate -> identify relationship -> gain access -> new identity/host -> enumerate again
```
