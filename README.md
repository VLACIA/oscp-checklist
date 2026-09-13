# OSCP Checklist

An Obsidian-based reference vault for working through OSCP-style standalone machines and Active Directory environments. It collects repeatable workflows, command references, attack decision trees, and visual canvases in one place for use during labs and exam preparation.

> [!WARNING]
> This material is intended for legal training environments and systems you are explicitly authorized to test. Commands and techniques can disrupt services or expose sensitive data; review and adapt them before use.

## Start here

1. Clone or download this repository.
2. Open the repository folder as a vault in [Obsidian](https://obsidian.md/).
3. Begin with one of the visual maps:
   - [`Standalone Box Method.canvas`](Standalone-Box.canvas) for Linux and Windows standalone targets.
   - [`ActiveDirectory.canvas`](./ActiveDirectory.canvas) for Active Directory attack paths.
4. Follow the linked notes, replacing placeholders such as `<TARGET_IP>`, `<LHOST>`, usernames, domains, and ports with values from your lab.

The Markdown files can also be read directly on GitHub, but Obsidian provides the intended canvas navigation and internal linking experience.

## Vault contents

| Area | What it covers |
| --- | --- |
| [Enumeration](./Enumeration-stand-alone/Plan.md) | Full-port scanning, Nmap, web, SMB, SNMP, and port-to-action references |
| [Standalone Linux](Plan-standalone-linux.md) | Initial access, common web attack methods, attack chains, and Linux privilege escalation |
| [Standalone Windows](Initial%20Access.md) | Windows footholds, credential dumping, token privileges, and privilege escalation |
| [Active Directory](./Active%20Directory/intro.md) | Enumeration, footholds, credential attacks, AD CS, lateral movement, persistence, MSSQL, and pivoting |
| [Password attacks](./Basics/Password%20Attacks/Intro.md) | Online attacks, cracking methodology/rules, KeePass and SSH-key cracking, NTLM/Net-NTLMv2, Pass-the-Hash, and relay |
| [Common CVEs and exploits](./Common%20CVEs%20%26%20Exploits/Intro.md) | Exploit research, Searchsploit, and frequently encountered vulnerabilities |
| [Shells and upgrades](./Shell%20%26%20upgrade/Intro.md) | Reverse-shell one-liners and Linux TTY upgrades |
| [File transfer](./File%20Transfer/Intro.md) | Practical transfer and exfiltration commands across Linux and Windows |
| [Definitions](./Definitions/Basic.md) | Supporting concepts, terminology, and Linux/Windows distinctions |

## Suggested workflow

Use the vault as a checklist rather than a script:

1. Enumerate every exposed service and record versions, credentials, and hypotheses.
2. Prioritize likely footholds and validate exploits against the exact target version.
3. Establish and stabilize a shell.
4. Transfer enumeration tools only when needed, then perform local privilege escalation.
5. In Active Directory, treat each host as part of a chain: enumerate, collect credentials, identify relationships, pivot, and repeat.
6. Keep evidence and commands organized as you work.

## Scope and limitations

- This is a personal study and field-reference project, not an official OffSec resource.
- The notes are concise by design and assume familiarity with basic networking, Linux, Windows, and command-line usage.
- Tool syntax and exam rules change. Verify commands against current upstream documentation and check the current exam guide before relying on any tool or technique.
- No checklist replaces understanding: inspect third-party exploit code and understand its impact before running it.

## Contributing

Corrections and focused additions are welcome. Keep notes practical, reproducible, and easy to scan; use placeholders instead of real credentials or target data. When adding or moving a note, update any affected canvas nodes and internal links.

## License

Released under the [MIT License](./LICENSE).
