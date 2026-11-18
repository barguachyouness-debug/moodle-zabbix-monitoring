# Security and repository publication notes

The project is an isolated educational lab. The controls below describe the intended security posture; review the actual firewall and service configuration before claiming that the whole VM is production-hardened.

## Controls implemented in the lab

- The Moodle agent listens on `192.168.56.10`, its host-only interface, rather than all interfaces.
- The agent's `Server` setting allows passive checks from `192.168.56.100`.
- The NGINX monitoring server binds to `127.0.0.1:8082`.
- NGINX status and PHP-FPM monitoring locations allow loopback access and deny other sources.
- PHP-FPM communicates over the local Unix socket `/run/php/php8.3-fpm.sock`.
- MariaDB monitoring uses a dedicated `zbx_monitor@localhost` account rather than Moodle's application database account.
- The Agent 2 MySQL plugin configuration was owned by `root:zabbix` with permissions `640`, allowing the agent to read the file without making it world-readable.
- The MariaDB plugin connects through `/run/mysqld/mysqld.sock`; there is no need to expose MariaDB port `3306` to the host-only network for this local monitoring design.

## Never commit these files or values

- `/etc/zabbix/zabbix_agent2.d/plugins.d/mysql.conf` containing the real password.
- Moodle's `config.php`.
- Database passwords, API tokens, `.env` files, or credential exports.
- SSH private keys, TLS private keys, certificate bundles containing private keys, or private WireGuard keys.
- MariaDB dumps, Moodle backups, `moodledata`, student records, uploaded assignments, or real user data.
- Unredacted logs, screenshots showing credentials, session tokens, private addresses beyond what you intend to publish, or personal information.

Use the `.example` files in `configs/` as sanitized templates. Replace all placeholders locally; never replace them with real secrets in a public commit.

## Before publishing

Run these checks from the repository root:

```bash
git status --short
git grep -n -I -E 'password\s*=|token\s*=|secret\s*=' -- ':!*.md'
find . -type f \( -name '*.key' -o -name '*.pem' -o -name '*.sql' -o -name '*.sql.gz' -o -name 'config.php' -o -name 'mysql.conf' \) -print
```

These are basic hygiene checks, not a complete secret scanner. Review every staged file and screenshot manually before pushing.
