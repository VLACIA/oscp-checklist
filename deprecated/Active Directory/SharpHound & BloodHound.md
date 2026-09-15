# SharpHound & BloodHound

Use BloodHound to turn AD objects and relationships into a graph and find **attack paths** that are difficult to see in raw command output.

> [!important]
> **Manual + automated enumeration work best together.** SharpHound can generate noticeable LDAP/API/registry traffic, so do not treat it as the only enumeration method. Use the manual techniques in [[Active Directory/PowerView & Manual AD Enumeration|PowerView & Manual AD Enumeration]] so you understand and can verify the graph.

## 1. Windows-side collection with SharpHound

Chapter 21 demonstrates the PowerShell ingestor:

```powershell
Import-Module .\SharpHound.ps1

# Show available options
Get-Help Invoke-BloodHound

# Collect the major relationship types and write a ZIP
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\<USER>\Desktop\ -OutputPrefix "corp audit"
```

`-CollectionMethod All` gathers the major relationships used for graph analysis, including groups, local-admin relationships, sessions/logged-on users, trusts, ACLs, containers, RDP, object properties, DCOM, SPN targets, and PowerShell remoting relationships.

SharpHound normally produces JSON data and zips it for transfer to the BloodHound analysis system. It may also create a cache file used to speed later collection.

### Snapshot vs looping

A normal run is a **snapshot**. Session state can change after the collection finishes, so a privileged user who logs on later may not appear. SharpHound also supports looping collection to catch changing session data.

For OSCP, the practical lesson is simpler: **re-collect when your identity or the environment meaningfully changes.**

## 2. Kali-side collection alternative

Your existing workflow can also collect directly from Kali when network reachability and DNS are correct:

```bash
bloodhound-python -u <USER> -p '<PASSWORD>' \
    -d corp.local -ns <DC_IP> -c All --zip
```

If Kali cannot directly reach the DC but a compromised Windows host can, prefer **Windows-side SharpHound collection** and transfer the ZIP back.

## 3. Start BloodHound / Neo4j

For the classic BloodHound setup used in PEN-200:

```bash
sudo neo4j start
bloodhound
```

Import the SharpHound ZIP into BloodHound.

## 4. First analysis queries

Useful starting points:

```text
Find all Domain Admins
Find Shortest Paths to Domain Admins
Find Shortest Paths to Domain Admins from Owned Principals
Find All Kerberoastable Users
Find AS-REP Roastable Users
Computers with Unconstrained Delegation
Find Principals with DCSync Rights
```

The graph is made of:

- **nodes** — users, groups, computers, domains, etc.
- **edges** — relationships such as membership, sessions, local-admin rights, ACL rights, and remote-access rights

Do not just look at the destination. Read every edge in the path and verify what it means.

## 5. Mark everything you control as Owned

This is one of the most important Chapter 21 habits.

Whenever you control a user or computer:

1. search for the object
2. right-click it
3. **Mark User as Owned** or **Mark Computer as Owned**
4. run **Shortest Paths to Domain Admins from Owned Principals**

Marking owned objects lets BloodHound calculate paths from your **actual current footholds** rather than from arbitrary principals.

```text
owned user/computer
      ↓
shortest path from owned principals
      ↓
edge you can abuse
      ↓
new identity / new host
      ↓
mark owned + collect / query again
```

## 6. Inspect edge details, not only the graph

For an interesting edge, open BloodHound's help/details. Chapter 21 highlights that BloodHound can provide:

- what the relationship means
- possible abuse methods
- OPSEC/detection considerations
- references

Examples of edges worth following up:

- `AdminTo` → current user is local admin on a computer
- session relationship → interesting user may have credentials/tokens on that computer
- `GenericAll`, `GenericWrite`, `WriteDACL`, etc. → ACL abuse
- group membership → inherited privilege

For ACL exploitation:

→ [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/ACL Abuse — Common BloodHound Paths|ACL Abuse — Common BloodHound Paths]]

For SPN-backed service accounts:

→ [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]

## 7. The Chapter 21 path-building mindset

A typical graph can reveal a chain such as:

```text
controlled low-priv user
    ↓ AdminTo
workstation
    ↓ privileged user's session
privileged user credentials/tokens
    ↓ MemberOf
Domain Admins
```

The value of BloodHound is not simply "find Domain Admin." It shows **which relationship to abuse next**.


## Chapter 24 — raw queries and session-driven targeting

Chapter 24 uses BloodHound not only for shortest paths, but also as a fast way to build an inventory and correlate **sessions + SPNs + hosts**.

### Basic object inventory with Cypher

```cypher
MATCH (m:Computer) RETURN m
MATCH (m:User) RETURN m
```

Use these to quickly list the computers/users SharpHound actually discovered before assuming your manual inventory is complete.

### Show active sessions

```cypher
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
```

Interpret sessions as targeting information:

```text
Domain Admin session on HOST
    ↓
HOST becomes high-value
    ↓
obtain local admin / SYSTEM on HOST
    ↓
inspect LSASS / cached credentials
```

A session displayed as a SID rather than a domain username can represent a **local** principal. RID `500` is the built-in local Administrator account, which can be important when reasoning about local-account reuse and relay paths.

### Chapter 24 pre-built checks

In addition to your normal shortest-path queries, check:

```text
Find all Domain Admins
Find Workstations where Domain Users can RDP
Find Servers where Domain Users can RDP
Find Computers where Domain Users are Local Admin
Shortest Path to Domain Admins from Owned Principals
List all Kerberoastable Accounts
```

No result is still useful: it rules out a class of easy movement and tells you to look at sessions, SPNs, services, ACLs, or application paths instead.

For Kerberoasting, inspect the SPN rather than only the username. A value like:

```text
http/<INTERNAL_HOST>.<DOMAIN>
```

links the account to a concrete internal service and may suggest where the cracked password will be useful.

`krbtgt` commonly appears as Kerberoastable, but Chapter 24 notes that its domain-generated password makes ordinary password cracking impractical; prioritize realistic service accounts instead.

### Do not stop enumeration because one vector appeared

If you find a roastable account, privileged session, or interesting ACL, **record it and finish enough enumeration to compare paths**. Chapter 24 explicitly continues service/SMB enumeration even after identifying a promising Kerberoast target.

### Groups and GPOs

Chapter 24 skips group/GPO enumeration only because those objects add no value in its lab scenario. In a real assessment, keep them in scope because they can reveal indirect privileges and policy-based attack paths.

Related: [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]] · [[Active Directory/After-escalation-before-movement|After escalation before movement]]

## 8. Re-run after every meaningful gain

After you obtain any of the following, update your graph and repeat analysis:

- a new domain user/password/hash/ticket
- local admin on another machine
- another shell
- a new reachable subnet
- a newly discovered session
- a newly controlled AD object

Return to:

→ [[Active Directory/Attack-Phases/Phase1-Enum|Phase 1 — Enumeration]]
