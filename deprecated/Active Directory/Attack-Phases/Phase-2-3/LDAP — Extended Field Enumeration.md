

```
# Check description AND info AND comment fields (all can have passwords):
ldapsearch -x -H ldap://192.168.x.100 \
    -D "stephanie@corp.local" -w 'Password123' \
    -b "DC=corp,DC=local" \
    "(objectClass=user)" sAMAccountName description info comment

# Also check computer accounts for descriptions:
ldapsearch -x -H ldap://192.168.x.100 \
    -D "stephanie@corp.local" -w 'Password123' \
    -b "DC=corp,DC=local" \
    "(objectClass=computer)" name description
```