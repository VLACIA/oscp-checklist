# Password Attacks — Decision Guide

Passwords are commonly protected with hashes or encryption rather than stored as plaintext. **Cracking** means generating candidate passwords and comparing their resulting representation with the captured/stored value; a cryptographic hash is not simply "decrypted."

## Choose the attack from the material you have

```text
live login service
  → online dictionary attack / password spray
  → noisy; check lockout and defensive controls first

hash / protected credential file
  → extract + format + crack offline
  → no account lockout and no repeated network authentication

NTLM hash
  → crack (mode 1000) OR Pass-the-Hash when the service/tool supports it

Net-NTLMv2 challenge-response
  → crack (mode 5600) OR relay when the environment permits it
```

## PEN-200 cracking methodology

1. **Extract** the hash or protected credential material.
2. **Format** it for the cracking tool and identify the hash type (`hashid`, `hash-identifier`, `*2john` helpers).
3. **Estimate feasibility**: cracking time is approximately `keyspace / hash rate`; `hashcat -b` benchmarks your hardware.
4. **Prepare the wordlist**: prefer target-informed mutations/rules over blindly trying huge lists.
5. **Attack the hash**, carefully verifying format and hash mode before spending time.

See [[Basics/Password Attacks/Cracking Methodology and Rules|Cracking Methodology and Rules]] for the detailed checklist.

## OSCP reminders

- `rockyou.txt` is a useful baseline, but password policy and **human password patterns** should guide mutations.
- Keep every recovered plaintext password: it may be useful for a controlled spray against other discovered accounts/services.
- Keep NTLM and Net-NTLMv2 conceptually separate: an **NTLM hash** can be passed; a **Net-NTLMv2 challenge-response** is normally cracked or relayed.
- If one cracking tool cannot handle a format/cipher, try another supported tool rather than assuming the credential material is unusable.
