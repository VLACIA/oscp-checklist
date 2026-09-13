# Chain C — LFI → SSH Private Key → Shell → Privilege Escalation

**Attack path:** vulnerable file parameter → read a user's SSH key → SSH login → unsafe `sudo` rule → root

## Brief explanation

1. **Identify local file inclusion (LFI):** A parameter such as `?file=` may pass an attacker-controlled path to a server-side file-reading function.
2. **Read useful local files:** After confirming access with a harmless file, the flaw may expose files such as `/etc/passwd` and `/home/<user>/.ssh/id_rsa` if the web process has permission to read them.
3. **Use the recovered key:** Save the private key, restrict its local permissions with `chmod 600`, and authenticate over SSH as the matching user. An encrypted key still requires its passphrase.
4. **Escalate privileges:** Check `sudo -l`. A `NOPASSWD` entry for a binary that can execute commands or access arbitrary files may be exploitable using the relevant GTFOBins technique.

**Why the chain works:** The web application exposes a private authentication secret, and an overly broad `sudo` policy converts user-level access into root access.
