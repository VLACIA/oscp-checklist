```
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.2   # Running processes
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2   # Installed software
snmpwalk -c public -v2c <TARGET_IP> 1.3.6.1.4.1.77.1.2.25    # Windows users
snmp-check <TARGET_IP> -c public
```