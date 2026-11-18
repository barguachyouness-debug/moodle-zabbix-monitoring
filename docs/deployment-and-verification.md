# Deployment and verification checklist

This guide records the key settings and tests used in the lab. It is a reproducibility aid, not a substitute for checking the current Zabbix 7.0 template documentation and the configuration already present on the target VM.

## 1. Lab addressing

| VM | Address | Role |
|---|---:|---|
| Zabbix appliance | `192.168.56.100` | Central monitoring server and frontend |
| Moodle server | `192.168.56.10` | Ubuntu Server 24.04, NGINX, PHP-FPM, MariaDB |

Both VMs use the VirtualBox host-only network `192.168.56.0/24`.

## 2. Zabbix Agent 2 on Moodle

Install Zabbix Agent 2 from the Zabbix 7.0 repository appropriate for Ubuntu 24.04. The lab used Agent 2 version `7.0.32`.

Core settings in `/etc/zabbix/zabbix_agent2.conf`:

```ini
Server=192.168.56.100
ServerActive=192.168.56.100
Hostname=Moodle-server
ListenIP=192.168.56.10
```

Validate and restart:

```bash
sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2
sudo systemctl restart zabbix-agent2
sudo systemctl status zabbix-agent2 --no-pager
```

On the Zabbix appliance, test the passive agent check:

```bash
zabbix_get -s 192.168.56.10 -p 10050 -k agent.ping
```

Expected result: `1`.

In the frontend, create or update host `Moodle-server`, set its Agent interface to `192.168.56.10:10050`, and link **Linux by Zabbix agent**.

## 3. NGINX status

Confirm NGINX includes the status module:

```bash
sudo nginx -V 2>&1 | grep -o 'with-http_stub_status_module'
```

The lab's local-only monitoring server in `/etc/nginx/conf.d/nginx-status.conf`:

```nginx
server {
    listen 127.0.0.1:8082;
    server_name localhost;

    location = /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }

    location ~ ^/(status|ping)$ {
        access_log off;
        allow 127.0.0.1;
        deny all;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Test locally on Moodle:

```bash
curl -i http://127.0.0.1:8082/nginx_status
```

Test through Agent 2 from the Zabbix appliance:

```bash
zabbix_get -s 192.168.56.10 -p 10050 \
  -k 'web.page.get[127.0.0.1,nginx_status,8082]'
```

The test returned HTTP `200 OK` and NGINX status counters in the lab. Link **Nginx by Zabbix agent** to `Moodle-server` and set the template's stub-status host, path, and port macros to match this listener and path.

## 4. PHP-FPM status and ping

The lab uses PHP 8.3-FPM, pool `www`, and socket `/run/php/php8.3-fpm.sock`.

In `/etc/php/8.3/fpm/pool.d/www.conf`, enable (do not comment out) the directives:

```ini
pm.status_path = /status
ping.path = /ping
```

Validate and reload PHP-FPM:

```bash
sudo php-fpm8.3 -t
sudo systemctl reload php8.3-fpm
sudo systemctl status php8.3-fpm --no-pager
```

The NGINX local monitoring server proxies `/status` and `/ping` to the PHP-FPM socket. Test locally:

```bash
curl -i http://127.0.0.1:8082/ping
curl -s 'http://127.0.0.1:8082/status?json'
```

Expected: `pong` from the ping endpoint and JSON pool statistics from the status endpoint.

Test through Agent 2 from the Zabbix appliance:

```bash
zabbix_get -s 192.168.56.10 -p 10050 \
  -k 'web.page.get[127.0.0.1,ping,8082]'

zabbix_get -s 192.168.56.10 -p 10050 \
  -k 'web.page.get[127.0.0.1,status?json,8082]'
```

The process name observed on the lab system was `php-fpm8.3`:

```bash
pid=$(systemctl show -p MainPID --value php8.3-fpm)
sudo awk '/^Name:/{print}' "/proc/$pid/status"
```

Link **PHP-FPM by Zabbix agent** to `Moodle-server`. Configure its endpoint and process-name macros to match the local paths, port, pool, and process name used by this installation. Confirm the actual macro names against the template revision installed in the frontend.

## 5. MariaDB through the Agent 2 MySQL plugin

Lab database details:

- MariaDB `10.11.14`
- Local socket `/run/mysqld/mysqld.sock`
- Monitoring account `zbx_monitor@localhost`
- Agent 2 named session `MoodleDB`

Create a dedicated monitoring user and grant only the privileges required by the Zabbix 7.0 MySQL template. The lab used:

```sql
CREATE USER 'zbx_monitor'@'localhost' IDENTIFIED BY 'SET_A_UNIQUE_LOCAL_SECRET';
GRANT REPLICATION CLIENT, PROCESS, SHOW DATABASES, SHOW VIEW
ON *.* TO 'zbx_monitor'@'localhost';
```

Do not reuse Moodle's database user. Do not put the actual password in this repository.

A local plugin configuration example is provided in [`../configs/mysql-plugin.conf.example`](../configs/mysql-plugin.conf.example). In the lab, the file was stored under `/etc/zabbix/zabbix_agent2.d/plugins.d/`, owned by `root:zabbix`, with mode `640` so the Zabbix agent could read it without making it world-readable.

Validate and restart Agent 2:

```bash
sudo zabbix_agent2 -T -c /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix-agent2
sudo systemctl status zabbix-agent2 --no-pager
```

From the Zabbix appliance, test the named session:

```bash
zabbix_get -s 192.168.56.10 -p 10050 -k 'mysql.ping[MoodleDB]'
zabbix_get -s 192.168.56.10 -p 10050 -k 'mysql.version[MoodleDB]'
```

Verified results in the lab were `1` and `10.11.14-MariaDB-0ubuntu0.24.04.1` respectively. Link **MySQL by Zabbix agent 2** and set its DSN macro to `MoodleDB`. The named session stores its credentials locally, so do not duplicate the password in the frontend unless the installed template's documentation specifically requires a different connection method.

## 6. Final frontend verification

In the Zabbix frontend:

1. Open **Data collection → Hosts → Moodle-server**.
2. Confirm the host Agent interface is `192.168.56.10:10050`.
3. Confirm the Linux, NGINX, PHP-FPM, and MySQL templates are linked.
4. Open **Monitoring → Latest data** and select `Moodle-server`.
5. Confirm recent items have values and are not unsupported.

Record a screenshot of the host's template list and separate Latest data views for NGINX, PHP-FPM, and MySQL.
