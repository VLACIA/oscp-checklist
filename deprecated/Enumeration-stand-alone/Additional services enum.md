# Additional service enumeration

Use the detected protocol—not the familiar port number—to choose tooling. Management products frequently move to high ports or hide behind HTTP(S).

## VNC (5900+)

```bash
nmap -sV -p5900-5910 --script vnc-info,vnc-title <TARGET_IP>
vncviewer <TARGET_IP>:0
```

Record the protocol/security type, desktop name, and whether authentication is required. Avoid password guessing unless explicitly permitted.

## Redis (6379)

```bash
nmap -sV -p6379 --script redis-info <TARGET_IP>
redis-cli -h <TARGET_IP> PING
redis-cli -h <TARGET_IP> INFO
```

Check authentication/ACLs, bind exposure, version, replication role, key names, and configuration access. Start with read-only commands; writes, replication changes, and module loading alter the target.

## Management and alternate web interfaces

- Probe every HTTP-like port (for example 8000, 8080, 8443, 8888, 9000) with the full web workflow.
- Identify product/version, virtual host, base path, API documentation, default pages, authentication realm, exposed metrics, backups, and default credentials where permitted.
- Common products include Tomcat, Jenkins, WebLogic, phpMyAdmin, Docker/Kubernetes dashboards, network-device panels, and bespoke admin APIs.
- Inspect TLS certificates and redirects for additional hostnames, then rescan any newly resolved hosts that are in scope.
