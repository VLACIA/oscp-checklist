```
# 1. MIME: Change Content-Type to image/jpeg
# 2. Magic bytes: GIF89a prepend
# 3. Double ext: shell.php.jpg, shell.php%00.jpg
# 4. Case: shell.pHp, shell.PHP
# 5. Alt ext: .php3 .php4 .php5 .phtml .phar .shtml
# 6. Null byte (older): shell.php%00.gif
```