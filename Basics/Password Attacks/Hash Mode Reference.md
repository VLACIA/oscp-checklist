# Hashcat Mode Reference

| Mode | Hash Type | Typical use in these notes |
| ---: | --- | --- |
| `0` | MD5 | Web/application hash example |
| `1000` | NTLM | Windows local/SAM hash |
| `5600` | NetNTLMv2 | Responder-captured challenge-response |
| `5500` | NetNTLMv1 | Older network challenge-response |
| `13400` | KeePass 1/2 | KeePass `.kdbx` master-password cracking |
| `22921` | OpenSSH private key `$6$` | PEN-200 SSH-key example; Hashcat compatibility can vary |
| `13100` | Kerberos TGS-REP | Kerberoasting |
| `18200` | Kerberos AS-REP | AS-REP roasting |
| `1800` | SHA512crypt `$6$` | Linux `/etc/shadow` |
| `500` | MD5crypt `$1$` | Older Linux shadow hashes |
| `3200` | bcrypt `$2y$` | Modern Linux/web password hashes |

```bash
# Discover likely type; confirm with source/context
hashid <HASH>
hash-identifier

# Search Hashcat modes
hashcat --help | grep -i '<TYPE>'

# Common examples
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 5600 netntlm.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1800 shadow.txt /usr/share/wordlists/rockyou.txt
```

> [!IMPORTANT]
> Hash-identification tools can return multiple possibilities. Confirm the hash using **where it came from**, its structure, and the cracking tool's expected format. A wrong mode wastes time.
