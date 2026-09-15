# Client-Side Attack Workflow

> **PEN-200 Chapter 11.** Client-side attacks are an initial-access path for systems that are not directly routable from the outside. Instead of exploiting an exposed service, the attack depends on a user opening a file or link that causes local client software to execute attacker-controlled content.

## Fast decision flow

```text
Identify likely user / role
        ↓
Recon target software + OS
        ├─ public document metadata
        └─ client/device fingerprinting
        ↓
Choose a compatible delivery path
        ├─ Microsoft Office available
        │      → [[Microsoft Office Macros]]
        └─ Windows client / alternate delivery
               → [[Windows Library-ms + LNK]]
        ↓
User opens / executes content
        ↓
Reverse shell / foothold
        ↓
normal Windows post-exploitation + privilege escalation
```

## 1. Recon before payload selection

Client-side payloads must match the target's **operating system and installed applications**. Start with passive information gathering when possible.

### Public document metadata

Find documents and inspect their metadata for employee names, document age, application/version, and OS clues:

```text
site:<TARGET_DOMAIN> filetype:pdf
```

```bash
exiftool -a -u <FILE>
```

See: [[StandAloneBoxes/Standalone Linux Box Methodology/Web-attacks-Methods/Exiftool — Metadata → Usernames|ExifTool — Metadata → Users / Software]]

Remember that old documents can provide stale information and different organizational branches may use different software.

## 2. Client / device fingerprinting

PEN-200 demonstrates **Canarytokens** using a **Web bug / URL token**. Send the generated tracking link as part of an authorized pretext. When the target opens it, the token can reveal information such as:

- source IP / location information
- browser
- operating system
- browser/device details gathered through JavaScript fingerprinting

The browser **User-Agent** can help infer OS/browser but can be modified and is therefore not fully reliable. PEN-200 notes that JavaScript-derived browser fingerprinting can provide more precise information.

Canarytokens can also be embedded in **Word documents, PDFs, and images** so opening/viewing the object generates a signal. The chapter also mentions **Grabify** and **fingerprint.js** as alternatives.

## 3. Pretext / delivery

A pretext gives the target a believable reason to open a file or link. Tailor it to the user's role and the information found during reconnaissance rather than sending arbitrary content.

Delivery mechanisms discussed in Chapter 11 include:

```text
email attachment
link to malicious / hosted content
USB drop
watering-hole style delivery
```

Email attachments and links may be inspected or filtered by spam filters, firewalls, and other security controls, so delivery method matters as much as payload choice.

## OSCP mental model

```text
No useful externally exposed attack?
        ↓
Can a user interaction path provide initial access?
        ↓
Recon user + software + OS
        ↓
Office macro OR Library-ms → LNK
        ↓
PowerShell / PowerCat
        ↓
foothold
```

Use client-side/social-engineering techniques only inside the engagement scope and authorized rules of engagement.
