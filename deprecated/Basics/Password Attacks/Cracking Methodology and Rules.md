# Cracking Methodology and Hashcat Rules

## Five-step cracking workflow

```text
1. Extract
2. Format / identify
3. Estimate feasibility
4. Prepare + mutate wordlist
5. Crack
```

### 1. Extract

Collect the original hash/file without modifying it. Examples in Chapter 13 include SAM NTLM hashes, KeePass `.kdbx` databases, SSH private keys, and captured Net-NTLMv2 challenge-responses.

### 2. Format and identify

```bash
hashid <HASH>
hash-identifier
hashcat --help | grep -i '<TYPE>'
```

Some protected files need conversion before cracking:

```bash
keepass2john Database.kdbx > keepass.hash
ssh2john id_rsa > ssh.hash
```

Watch for helper-script prefixes such as `Database:` or `id_rsa:` when Hashcat expects only the hash body.

### 3. Estimate feasibility

```text
cracking time ≈ keyspace / hash rate
keyspace = charset_size ^ password_length
```

```bash
hashcat -b
```

GPU cracking is usually much faster for suitable algorithms. Password **length** grows the keyspace dramatically, so do not burn exam time on unrealistic brute force when another path exists.

### 4. Prepare and mutate the wordlist

Password policies often cause predictable human mutations: capitalize the first character, append digits, then append a common special character.

Useful Hashcat rule functions from PEN-200:

| Rule | Effect |
| --- | --- |
| `$1` | append `1` |
| `$!` | append `!` |
| `^3` | prepend `3` |
| `c` | capitalize first character and lowercase the rest |

Rule functions on the **same line** are applied consecutively to one candidate. Rules on **separate lines** create separate mutations.

Example:

```text
# demo.rule
$1 c $!
$2 c $!
$1 $2 $3 c $!
```

Preview candidates without cracking:

```bash
hashcat -r demo.rule --stdout wordlist.txt
```

Built-in Hashcat rules:

```bash
ls -la /usr/share/hashcat/rules/
```

Useful Chapter 13 examples:

```text
best64.rule
rockyou-30000.rule
```

### 5. Crack

```bash
hashcat -m <MODE> hashes.txt wordlist.txt -r <RULE_FILE>
```

## Exam-time decision

```text
known password policy / user habits?
  YES → create a small targeted wordlist + focused rules
  NO  → start with a proven wordlist + built-in rules

estimated cracking time unreasonable?
  YES → pursue another attack vector in parallel
```
