# Troubleshooting notes from the lab

This file records real issues encountered while integrating the Moodle services with Zabbix.

## Agent reports the wrong hostname

**Symptom:** Agent log reports `Zabbix server` even though it is running on the Moodle VM.

**Cause:** The agent's `Hostname` setting was still at its default.

**Resolution:** Set `Hostname=Moodle-server` in `/etc/zabbix/zabbix_agent2.conf`, restart the agent, and use the same exact name in the Zabbix frontend.

## NGINX status endpoint returns 404

**Symptom:** `/basic_status` returns HTTP 404.

**Cause:** The configured NGINX status location was `location = /nginx_status`, not `/basic_status`.

**Resolution:** Test the configured path:

```bash
curl -i http://127.0.0.1:8082/nginx_status
```

## NGINX configuration test fails because locations are nested

**Symptom:** NGINX reports that a location cannot be inside the exact `/nginx_status` location.

**Cause:** The PHP-FPM location block was accidentally pasted inside the NGINX exact-match location.

**Resolution:** Place both `location` blocks as sibling locations inside the same `server {}` block, then run `sudo nginx -t` before reloading.

## PHP-FPM monitoring endpoints are missing

**Symptom:** Grep output shows `;pm.status_path` or `;ping.path`.

**Cause:** A leading semicolon comments out the directive in the PHP-FPM pool configuration.

**Resolution:** Enable the directives in the correct pool file:

```ini
pm.status_path = /status
ping.path = /ping
```

Validate with `sudo php-fpm8.3 -t`, reload PHP-FPM, and test the local endpoints.

## MariaDB plugin returns access denied (1045)

**Symptom:** `mysql.ping[MoodleDB]` returns `0` or `mysql.version[MoodleDB]` reports MariaDB error 1045.

**Cause:** The password configured in MariaDB did not match the password in the Agent 2 plugin file.

**Resolution:** Synchronize the same locally stored password in both places, then restart Agent 2 and test `mysql.ping[MoodleDB]` and `mysql.version[MoodleDB]` again. Never paste the password into tickets, chat, GitHub, or screenshots.

## NGINX/PHP-FPM local metrics work but Latest data is empty

**Checks:**

1. Confirm the correct template is linked to `Moodle-server`.
2. Confirm the template macros match the actual local status host, path, and port.
3. Inspect the item state and error in Zabbix; do not assume a missing search result means the template is absent.
4. Search Latest data by an item name such as `stub status` or `Get stub status page`, not only by the word `Nginx`.
5. Test the same item key directly with `zabbix_get` where applicable.
