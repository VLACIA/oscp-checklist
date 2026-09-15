# RPC and NFS enumeration (111/2049 TCP/UDP)

## Discover RPC services and exports

```bash
rpcinfo -p <TARGET_IP>
nmap -sV -p111,2049 --script rpcinfo,nfs-showmount,nfs-ls,nfs-statfs <TARGET_IP> -oN nfs.txt
showmount -e <TARGET_IP>
```

Remember that RPC may assign important services to high dynamic ports; feed discovered ports back into targeted Nmap scans.

## Mount and inspect

```bash
mkdir -p /tmp/nfs-mount
sudo mount -t nfs -o ro,nolock <TARGET_IP>:/<EXPORT> /tmp/nfs-mount
find /tmp/nfs-mount -maxdepth 3 -ls
sudo umount /tmp/nfs-mount
```

- Start read-only. Record export restrictions, ownership (numeric UID/GID), permissions, hidden files, keys, configs, backups, and web content.
- Check NFS versions and whether `root_squash`/`no_root_squash` behavior is relevant.
- Do not modify an export until its purpose and impact are understood.
