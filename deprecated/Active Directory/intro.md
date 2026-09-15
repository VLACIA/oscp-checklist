# Active Directory — OSCP Mental Model

==**AD in plain English**==

**Active Directory (AD) is the identity and management layer for a Windows domain.** It stores and organizes **users, groups, computers, permissions, and other objects**. A domain can have **one or more Domain Controllers (DCs)**; the DCs are the core of the domain and store AD objects and attributes.

AD depends heavily on **DNS**. In a typical domain, a DC also provides or works closely with authoritative DNS so clients can find domain services. Objects are commonly organized into **Organizational Units (OUs)**, which act like containers for users, computers, and other AD objects.

A single AD environment can contain multiple domains. Domains can form **trees**, and multiple trees can form a **forest**. **Domain Admins** are highly privileged inside their domain; **Enterprise Admins** have forest-wide control and administrator privileges on all DCs in the forest.

## The attacker's view

AD is valuable because it contains both **objects** and the **relationships/permissions between those objects**. A low-privileged domain user may still be able to read large portions of the directory and discover:

- users and privileged groups
- computers, servers, and operating systems
- where the current user has local administrator access
- where interesting users are logged on
- service accounts and SPNs
- dangerous ACL relationships
- readable SMB shares, SYSVOL, scripts, backups, and credentials
- paths that connect the identity you control to a higher-privileged identity

The important idea is not simply **"get Domain Admin immediately."** Build a map of the domain, identify relationships, and chain them together. A user that looks equivalent to your current user may still have a unique permission somewhere else.

> [!important] Rinse and repeat
> **Every new user, computer, shell, or credential changes your enumeration viewpoint.** Re-run the important enumeration steps after each new identity or foothold. Permissions in AD are complex, and a new low-privileged user may expose a path the previous user could not see.

## AD enumeration workflow

Use this as the Chapter 21 mental model:

**valid domain identity → users/groups/computers → local-admin access → logged-on sessions → SPNs → ACLs → shares/SYSVOL → SharpHound/BloodHound → mark owned → identify attack path → gain a new identity → repeat enumeration**

Start here: [[Active Directory/Attack-Phases/Phase1-Enum|Phase 1 — AD Enumeration]].

For Windows-side manual enumeration and PowerView commands: [[Active Directory/PowerView & Manual AD Enumeration|PowerView & Manual AD Enumeration]].

For graph collection and analysis: [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]].

## LDAP / object naming — minimum you should know

Most AD enumeration ultimately relies on **LDAP**. A Distinguished Name (DN) uniquely identifies an object in the directory.

```text
CN=Stephanie,CN=Users,DC=corp,DC=local
```

- `CN=` = Common Name / object or container name
- `DC=` = Domain Component
- base domain `corp.local` → `DC=corp,DC=local`
- narrowing the LDAP search base to a container limits what you search

Useful Windows checks:

```powershell
# Current domain / PDC information
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Domain DN in LDAP format
([adsi]'').distinguishedName
```

The **PdcRoleOwner** is useful when you want a specific DC for consistent LDAP queries.

---

==**Kerberos in plain English — read this before the "roasting" attacks**==

**Kerberos is how Windows logs you in without passing your password around.** Picture a cinema: you show ID once at the entrance and get a wristband (a _ticket_). After that, every screen lets you in by checking the wristband — you never show ID again.

**Why hackers care:** some tickets or authentication exchanges are protected using material derived from an account password. When the protocol allows us to obtain crackable material, we can test password guesses offline. That is the basis of **Kerberoasting** and **AS-REP Roasting**.

---

==How the 40 AD points actually work==

**The AD set is 3 chained machines worth 40 points total** — the highest-value and most predictable part of the exam. Points are banked as you progress: **MS01 local.txt = 10**, **MS02 local.txt = 10**, **DC proof.txt = 20**. So even if you don't fully own the DC, reaching MS01 and MS02 still banks 20 points.

**The one insight that makes AD click:** it's a **chain**, not three separate boxes. The credentials and hashes you loot on one machine are the keys to the next. Every phase is the same rhythm — enumerate → find a quick win → get a foothold → loot creds → reuse them to move forward → re-enumerate from the new identity → finally reach Domain Admin and own the DC.
