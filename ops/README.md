# Server ops

Config for the production server (Ubuntu 20.04, Apache 2.4 prefork, mod_php 7.4). These files aren't used by the app; they're copied onto the server by hand.

## Apache snippets (`apache/`)

Each file is a standalone `conf-available` snippet, so it can be switched on or off independently.

| File | What it does |
|---|---|
| `tciaf-logformat.conf` | Adds request duration in microseconds (`%D`) to the end of the `combined` access-log format |
| `tciaf-status.conf` | Loopback-only vhost on `127.0.0.1:8081`, so `/server-status` isn't rewritten to `index.php` |
| `tciaf-timeouts.conf` | `KeepAliveTimeout 2`, `Timeout 30`, so idle or stalled clients don't hold one of the 10 prefork workers |
| `tciaf-block-ua.conf` | 403s high-volume crawlers by User-Agent (`/robots.txt` stays reachable) |
| `tciaf-block-probes.conf` | 403s vulnerability-scanner paths (dotfiles, `.env`/`.bak`/…, `wp-*`, and similar) |
| `tciaf-static-cache.conf` | `Cache-Control: public, max-age=604800` on static images, css and fonts (not js; `/js/bloc.js` has no version string) |

Install or update:
```sh
sudo install -m 644 ops/apache/tciaf-*.conf /etc/apache2/conf-available/
sudo a2enmod headers
sudo a2enconf tciaf-logformat tciaf-status tciaf-timeouts tciaf-block-ua tciaf-block-probes tciaf-static-cache
sudo apachectl configtest && sudo systemctl reload apache2 && systemctl is-active apache2
```

If the site's vhost still has the old `<filesMatch ".(ico|pdf|jpg|jpeg|png|svg|gif|js|css)$">` cache block from an earlier `setup.md`, remove it: vhost-level `Header set` runs after this server-level snippet, so the old block would win and keep caching `/js/bloc.js`.

Roll back one snippet with `sudo a2disconf <name> && sudo systemctl reload apache2`.

**Don't** write blocks as `<Location "/">` + `Require all granted`. That replaces existing denies elsewhere in the config (for example `.phps` source and `server-status`). Use `<If …> Require all denied </If>`, which only adds a deny.

## Incident snapshot (`bin/tciaf-snapshot`)

```sh
sudo install -m 755 ops/bin/tciaf-snapshot /usr/local/bin/tciaf-snapshot
sudo tciaf-snapshot    # writes /var/log/tciaf-snapshots/<UTC timestamp>.txt
```
Run it while the site is slow. It records memory, the Apache worker scoreboard (via `tciaf-status.conf`), the biggest workers, and connections per client IP.

## Response-time percentiles

Only lines in the new format end in a number:
```sh
sudo awk '$NF ~ /^[0-9]+$/ {print $NF/1000}' /var/log/apache2/access.log | sort -n \
 | awk '{a[NR]=$1} END {print "p50",a[int(NR*.5)],"ms  p95",a[int(NR*.95)],"ms  p99",a[int(NR*.99)],"ms  n",NR}'
```
