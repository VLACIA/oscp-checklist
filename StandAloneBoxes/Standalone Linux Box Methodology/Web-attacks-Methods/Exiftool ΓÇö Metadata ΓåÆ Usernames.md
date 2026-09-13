# ExifTool — Metadata → Users / Software

> **PEN-200 Chapter 11.** Public documents can reveal usernames, document age, operating-system clues, and the applications used inside a target organization. This is useful before choosing a client-side attack.

## Find public documents

Passive search:

```text
site:<TARGET_DOMAIN> filetype:pdf
site:<TARGET_DOMAIN> filetype:docx
```

If you are actively enumerating the target web server, search for document extensions with your normal content-discovery workflow, for example:

```bash
gobuster dir -u http://<TARGET>/ -w <WORDLIST> -x pdf,doc,docx,xls,xlsx,ppt,pptx
```

Active enumeration is noisier and can create target-side log entries.

## Inspect metadata

```bash
exiftool -a -u <FILE>
```

`-a` shows duplicate tags and `-u` shows unknown tags.

Focus on:

```text
Author / Creator       → internal employee / username lead
Create Date            → how current the intelligence is
Modify Date            → how current the intelligence is
Producer / CreatorTool → Office/application + version clues
OS-related metadata    → Windows/macOS clues
```

Quick sweep across collected files:

```bash
exiftool *.jpg *.docx *.pdf 2>/dev/null | grep -i "author\|creator\|owner\|producer\|create date\|modify date"
```

## How to use the result

```text
public document
      ↓
metadata
      ├─ employee / username → credential or social-engineering lead
      ├─ Microsoft Office    → consider Office client-side paths
      └─ OS/application clue → choose a payload compatible with the target
```

Do **not** treat metadata as guaranteed current truth: older documents may be stale, and different branches of the same organization may use different software.

Related: [[StandAloneBoxes/Standalone Win Box Methodology/Client-Side Attacks/Client-Side Attack Workflow|Client-Side Attack Workflow]]
