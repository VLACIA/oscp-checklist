```
# Check:
SELECT user, file_priv FROM mysql.user;
SHOW variables LIKE 'plugin_dir';
# Compile malicious UDF .so → upload → CREATE FUNCTION → system()
# If MySQL local-only → SSH tunnel first:
ssh -L 3306:127.0.0.1:3306 user@<TARGET_IP>
mysql -u root -h 127.0.0.1 -P 3306 -p
```