# Chain F — Exposed `.git` → Historical Credentials → SSH → `sudo`

**Attack path:** public `.git` metadata → reconstruct repository → inspect history → recover deleted secret → SSH login → root

## Brief explanation

1. **Detect repository exposure:** If `/.git/HEAD` or other Git metadata is reachable through the web server, the application's repository may be downloadable even when directory listing is disabled.
2. **Reconstruct the repository:** Tools such as `git-dumper` retrieve exposed Git objects so the project and its history can be inspected locally.
3. **Search commits and diffs:** Removing a password from the current file does not erase it from earlier commits. `git log`, `git show`, and diffs may reveal historical secrets.
4. **Test credential reuse:** A recovered application or deployment password may authenticate a system user over SSH.
5. **Escalate privileges:** Enumerate `sudo -l` and determine whether an allowed command can safely be converted into root command execution.

**Why the chain works:** Deployment accidentally publishes version-control history, secret rotation or history cleanup was not performed, and credential reuse plus a weak `sudo` policy completes the compromise.
