| Mode  | Hash Type          | Use Case             |
| ----- | ------------------ | -------------------- |
| 1000  | NTLM               | Windows local hashes |
| 5600  | NetNTLMv2          | Responder captures   |
| 5500  | NetNTLMv1          | Older captures       |
| 13100 | Kerberoast TGS-REP | SPN hash crack       |
| 18200 | AS-REP Roast       | AS-REP hash crack    |
| 1800  | SHA512crypt $6$    | Linux /etc/shadow    |
| 500   | MD5crypt $1$       | Linux shadow (older) |
| 3200  | bcrypt $2y$        | Modern Linux/web     |
| 0     | MD5                | Web app hashes       |
```
hashcat -m 1000 ntlm.txt rockyou.txt
hashcat -m 5600 netntlm.txt rockyou.txt
hashcat -m 13100 tgs.txt rockyou.txt -r rules/best64.rule
hashcat -m 18200 asrep.txt rockyou.txt
hashcat -m 1800 shadow.txt rockyou.txt
```