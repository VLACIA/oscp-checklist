# Chain D — Weak Web Login → File Upload → Web Shell → Cron Privilege Escalation

**Attack path:** default credentials → unsafe upload → executable web shell → shell as `www-data` → writable root-run script → root

## Brief explanation

1. **Test authorized default credentials:** Administrative interfaces are sometimes left with documented defaults such as `admin:admin`.
2. **Assess the upload control:** A weak uploader may validate only the filename extension or MIME type. Alternate executable PHP extensions such as `.phtml` may bypass incomplete filters when the server is configured to execute them.
3. **Trigger the uploaded file:** If the upload directory is web-accessible and allows script execution, requesting the file can provide command execution as the web-service account.
4. **Inspect scheduled tasks:** Review `/etc/crontab` and related cron definitions. If root periodically executes a script that the compromised account can modify, changing that script can cause commands to run as root.

**Why the chain works:** Weak authentication exposes a dangerous upload feature, the server executes user-controlled content, and a permissions error lets an unprivileged user alter a root-run cron task.
