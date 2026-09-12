#### Archive / ZIP Cracking

```
zip2john protected.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
office2john file.xlsx > office.hash
john office.hash --wordlist=rockyou.txt
binwalk -e suspicious_file    # extract embedded files
```