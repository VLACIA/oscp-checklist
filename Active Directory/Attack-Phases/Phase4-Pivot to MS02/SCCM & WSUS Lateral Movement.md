Both are management infrastructure that pushes software/updates to every domain machine — if you can talk to either as a low-priv user, you can often push a "package" that's actually your reverse shell, landing as SYSTEM on any managed box.

```
# SCCM — check client namespace access, then look for policy/NAA creds:
Get-WmiObject -Namespace "root\ccm\clientsdk" -Class CCM_Application
nxc smb <SCCM_server> -u stephanie -p 'Password123' --shares
# SharpSCCM / sccmhunter for full recon + NAA credential theft

# WSUS — unsigned HTTP updates can be swapped for malicious ones:
Get-WSUSServer   # confirm it's plain http:// not https://
nxc smb <WSUS_server> -u stephanie -p 'Password123' --shares
# WSUS-based MITM tools (e.g. pywsus/WSUSpect) inject a malicious "update"
```