# Antivirus Evasion Workflow

> **PEN-200 Chapter 12.** Use this branch when a Windows payload is blocked or quarantined by antivirus. The chapter separates evasion into **on-disk** and **in-memory** approaches and emphasizes targeting the **specific AV product/configuration** instead of trying to create a universal bypass.

## Fast decision flow

```text
Payload/file blocked or quarantined
        ↓
Identify target AV + confirm protection is active
        ↓
[[AV Detection and Testing|Reproduce/test against that AV safely]]
        ↓
Choose an evasion direction
        ├─ On-disk
        │    ├─ packer
        │    ├─ obfuscator
        │    └─ crypter
        │
        └─ In-memory
             ├─ process injection
             ├─ reflective DLL injection
             ├─ process hollowing
             └─ inline hooking
        ↓
Need a practical PEN-200 path?
        ├─ PowerShell → [[PowerShell Thread Injection]]
        └─ Automated PE injection → [[Shellter]]
        ↓
Retest against target-like AV configuration
        ↓
Execute → catch shell → continue normal Windows post-exploitation
```

## 1. Know what the AV can inspect

Modern AV can use several engines at the same time:

```text
File engine      → files on disk / real-time file activity
Memory engine    → process memory + suspicious API use
Network engine   → inbound/outbound traffic, including C2 patterns
Disassembler     → reconstruct / inspect machine code
Sandbox/emulator → safely execute and analyze suspicious code
Browser plugin   → browser-visible malicious content
ML engine        → classify unknown/suspicious samples
```

Detection methods to remember:

- **Signature-based** — hashes, byte patterns, strings, or other known indicators.
- **Heuristic-based** — rules/algorithms inspect instructions, calls, and suspicious patterns.
- **Behavior-based** — execute/analyze the sample and look for malicious behavior.
- **Machine-learning** — analyze metadata/patterns to classify known or unknown threats.

A one-bit change produces a different cryptographic hash, which shows why **hash-only detection is fragile**. Modern AV therefore combines several detection methods.

### YARA

**YARA** is a signature language used to describe/match malware characteristics. A signature can target different aspects of the same malware, such as the file on disk or network behavior.

## 2. AV vs EDR vs SIEM

```text
AV  → prevent / detect / remove malicious software
EDR → endpoint telemetry + detection/response visibility
SIEM → central collection/correlation of security events
```

AV and EDR are not mutually exclusive. A file may bypass AV but **EDR can still alert the SOC on suspicious behavior**, especially memory injection or PowerShell activity.

## 3. On-disk evasion

Use when the payload must exist as a file on disk.

### Packers

Packers transform an executable into a functionally equivalent file with a different binary structure/hash. Simple/popular packers such as **UPX alone are not sufficient against modern AV**.

### Obfuscators

Obfuscation changes code structure without changing its purpose, for example:

- replace instructions with semantically equivalent ones
- insert irrelevant/dead code
- split or reorder functions

This can hinder reverse engineering and may affect signature matching.

### Crypters

A crypter stores executable code **encrypted on disk** and adds a decryption stub that restores the original code **in memory** at runtime.

Advanced protectors may combine packing/obfuscation/encryption with anti-reversing, anti-debugging, and VM/emulation-detection techniques.

## 4. In-memory evasion

In-memory approaches avoid writing the malicious payload to disk, reducing exposure to file-engine scanning.

### Remote Process Memory Injection

Classic Windows API flow:

```text
OpenProcess
    ↓
VirtualAllocEx
    ↓
WriteProcessMemory
    ↓
CreateRemoteThread
    ↓
payload executes in target process
```

- `OpenProcess` → obtain a handle to a process we can access.
- `VirtualAllocEx` → allocate memory inside that remote process.
- `WriteProcessMemory` → copy payload bytes into the allocated memory.
- `CreateRemoteThread` → start execution of the payload in the remote process.

### Reflective DLL Injection

Load a DLL directly from memory rather than loading a DLL from disk with `LoadLibrary`. Because normal Windows loading APIs expect a disk-backed DLL, reflective loading requires custom loading logic.

### Process Hollowing

```text
start legitimate process suspended
        ↓
remove/replace its in-memory image
        ↓
insert malicious executable image
        ↓
resume process
```

### Inline Hooking

Modify a function in memory so execution jumps to attacker-controlled code, then returns to the original function flow.

## OSCP memory aid

```text
AV blocks payload
    ↓
Do not immediately keep regenerating random payloads
    ↓
identify AV → reproduce it → choose disk vs memory → modify/test → execute
```

See also:

- [[AV Detection and Testing]]
- [[PowerShell Thread Injection]]
- [[Shellter]]
- [[StandAloneBoxes/Standalone Win Box Methodology/Initial Access|Windows Initial Access]]

tag:#windows tag:#av-evasion tag:#initial-access
