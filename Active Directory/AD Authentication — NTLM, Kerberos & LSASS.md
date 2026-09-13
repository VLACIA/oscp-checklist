# AD Authentication — NTLM, Kerberos & LSASS

> [!summary] Why this matters
> Chapter 22 attack techniques make much more sense if you understand **what is encrypted with which key** and **where Windows caches credentials/tickets**.

## NTLM authentication

NTLM uses **challenge-response**, not Kerberos tickets.

Common cases where NTLM may be used:
- Connecting to a server by **IP address** instead of hostname.
- Authenticating to a hostname that is not registered in AD-integrated DNS.
- Third-party applications that explicitly use NTLM.

High-level flow:

```text
password
  ↓
NTLM hash
  ↓
client → username → server
client ← challenge/nonce ← server
client → response(challenge encrypted with NTLM hash) → server
server → username + challenge + response → DC
DC computes expected response with stored NTLM hash
  ↓
match = authenticated
```

**Key point:** the password is not sent across the network. The DC validates the challenge-response using the user's NTLM hash.

NTLM hashes are fast to test offline, so weak passwords may be crackable.

---

## Kerberos authentication

Kerberos is the default AD authentication protocol and uses **tickets**. The Domain Controller acts as the **Key Distribution Center (KDC)**.

### 1. User authentication — AS-REQ / AS-REP

```text
Client                         DC / KDC
  |                               |
  | -------- AS-REQ ------------> |
  |   username + timestamp        |
  |   encrypted with key derived  |
  |   from user's password        |
  |                               |
  | <------- AS-REP ------------- |
  |   session key + TGT           |
```

The DC looks up the user's password hash in `ntds.dit` and validates the encrypted timestamp.

The **AS-REP** contains:
- A session key encrypted using the user's password-derived key.
- A **Ticket Granting Ticket (TGT)**.

The TGT contains user/domain/session information and is encrypted with the secret key of the **krbtgt** account, so the client cannot modify it.

> [!important]
> Missing Kerberos preauthentication is what enables [[Active Directory/Attack-Phases/Phase-2-3/AS-REP Roasting|AS-REP Roasting]].

### 2. Requesting access to a service — TGS-REQ / TGS-REP

When the user wants a domain resource such as SMB, LDAP, HTTP, or another SPN-backed service:

```text
Client                         DC / KDC
  |                               |
  | -------- TGS-REQ -----------> |
  |   TGT + target service/SPN    |
  |                               |
  | <------- TGS-REP ------------ |
  |   service session key         |
  |   + service ticket            |
```

The **service ticket** contains the user's identity/group membership and a session key. It is encrypted using the **password hash of the account running the target SPN/service**.

> [!important]
> This is the basis of [[Active Directory/Attack-Phases/Phase-2-3/Kerberoasting|Kerberoasting]]: request a service ticket, then crack the service-account encrypted material offline.

### 3. Authenticating to the application — AP-REQ

```text
Client ---------------- AP-REQ ----------------> Service
           service ticket + authenticator
```

The service decrypts the ticket with its service-account key, verifies the request, and uses the group memberships in the ticket when deciding access.

> [!important]
> If you know the service-account hash, this trust model can be abused with a [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/Silver Ticket & Diamond Ticket|Silver Ticket]].

---

## TGT vs TGS / service ticket

```text
TGT
├─ issued by KDC after user authentication
├─ encrypted with krbtgt key
└─ can be used to request additional service tickets

TGS / service ticket
├─ issued for a specific SPN/service
├─ encrypted with target service-account key
└─ normally useful only for that service/resource
```

This distinction matters when stealing cached tickets:
- **Steal a TGS** → access the resource associated with that ticket.
- **Steal a TGT** → request additional TGS tickets for resources the user can access.

See [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/pass-the-ticket|Pass-the-Ticket]].

---

## LSASS — cached AD credentials and tickets

Modern Windows stores reusable authentication material in **LSASS** memory to support Single Sign-On and ticket renewal.

Possible loot includes:
- NTLM hashes / cached authentication material.
- Kerberos TGTs.
- Kerberos service tickets.

LSASS runs as `SYSTEM`, so accessing another user's cached credentials normally requires **local Administrator / SYSTEM-level access**.

### Mimikatz

```text
privilege::debug
sekurlsa::logonpasswords
sekurlsa::tickets
```

`sekurlsa::logonpasswords` can expose credential material for logged-on users, including remote/RDP sessions.

`sekurlsa::tickets` lists cached Kerberos tickets.

> [!note]
> LSA Protection can prevent direct reads of LSASS memory. Chapter 22 also notes that Mimikatz is heavily signatured; LSASS can instead be dumped and analyzed offline.

See [[Active Directory/Win-MS01|Win-MS01 post-exploitation]].

---

## Quick attack mapping

```text
Missing pre-authentication
    → AS-REP Roasting

User/service account has SPN
    → Kerberoasting

Local admin / SYSTEM on workstation
    → LSASS hashes + tickets
    → crack / PtH / PtT

Service-account hash + Domain SID + target SPN
    → Silver Ticket

Replication rights
    → DCSync
```
