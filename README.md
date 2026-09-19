# nextcloud

State + restore docs for the **Nextcloud stack on the RU relay server** (plain HTTP by raw IP, co-hosted with the xray VPN relay — see the `vpn` repo for that side).

Server: Ubuntu **26.04.1**, kernel 7.0.0-31-generic. Stack: apache2 + php8.5-fpm + postgresql + redis + APCu.

| Component | Version |
|---|---|
| Nextcloud | 32.0.3.2 |
| PHP | 8.5.x (php8.5-fpm, unix socket `/run/php/php8.5-fpm.sock`) |
| PostgreSQL | system package, db `nextcloud`, user `nextclouduser` |
| Cache / locking | APCu (`memcache.local`) + Redis (`memcache.locking`, localhost:6379) |
| Apache | `:80` Nextcloud vhost bound to raw IP; stock `default-ssl` on `:8443` (snakeoil, unused) |

Repos are **public — secrets sanitized**. Real values (db password, instanceid, salts, secret) live only in `/var/www/nextcloud/config/config.php` on the server.

## Files

```
apache/nextcloud.conf        → /etc/apache2/sites-enabled/nextcloud.conf
apache/ports.conf            → /etc/apache2/ports.conf
php/www.conf                 → /etc/php/8.5/fpm/pool.d/www.conf (effective settings only)
config/config.php.example    → /var/www/nextcloud/config/config.php (fill secrets)
patches/versioncheck-php85.patch → see below
```

## The PHP 8.5 gate patch (IMPORTANT — re-apply after every Nextcloud update)

Ubuntu 26.04 ships **only PHP 8.5** (no ondrej PPA builds for 26.04), while Nextcloud 32 hard-caps `PHP_VERSION_ID >= 80500` → refuses to run. Patch (backup kept at `/root/versioncheck.php.bak`):

```diff
--- a/lib/versioncheck.php
+++ b/lib/versioncheck.php
@@ -17 +17 @@
-if (PHP_VERSION_ID >= 80500) {
+if (PHP_VERSION_ID >= 80600) {
```

Apply: `sed -i 's/PHP_VERSION_ID >= 80500/PHP_VERSION_ID >= 80600/' /var/www/nextcloud/lib/versioncheck.php`

⚠️ **`apt`/updater upgrades of Nextcloud overwrite this file** — if Nextcloud suddenly 500s/blank-pages after an update, re-apply the sed. Plan a Nextcloud 33+ upgrade (supports 8.5 officially) to retire the patch.

## Post-upgrade DB maintenance (26.04 → newer postgres collation)

```bash
sudo -u postgres psql -d nextcloud -c 'REFRESH COLLATION VERSION;'
sudo -u postgres reindexdb -d nextcloud
```

## Restore / rebuild guide

1. Ubuntu 26.04 + packages: `apache2 libapache2-mod-php8.5 php8.5-{fpm,pgsql,curl,gd,xml,mbstring,zip,intl,apcu} postgresql redis-server php-redis`.
2. Copy this repo's apache/php files into place; `a2enmod php8.5 rewrite headers env dir mime setenvif; a2ensite nextcloud; a2dissite 000-default` (keep 8443 ssl default if desired).
3. Drop the Nextcloud webroot + `data/` from backup (or install 32.0.3 and restore `data/` + `config/config.php` from `config.php.example` with real secrets).
4. `chown -R www-data:www-data /var/www/nextcloud`.
5. Restore postgres: `sudo -u postgres pg_restore -d nextcloud <dump>` (or full cluster `pg_restore`/`pg_upgrade` from the old host).
6. Re-apply the versioncheck patch above. Check `curl http://<RELAY_IP>/status.php` → `"installed":true,"maintenance":false`.
7. `php occ db:add-missing-indices` if status complains; REFRESH COLLATION VERSION + reindex per above.

## Lessons from the 26.04 upgrade (2026-09)

- **Check disk first** — the dist-upgrade stalled with 0 bytes free; a hung `redis-server.postinst` held the dpkg lock for 8.5h (`dpkg --audit`, `fuser -v /var/lib/dpkg/lock-frontend` to find it).
- Noble leftovers in sources.list break the 26.04 apt run — purge before full-upgrade.
- The upgrade removes php8.3-fpm and installs nothing in its place → Nextcloud down until the php8.5 stack is installed + vhost/socket paths updated.
- `REFRESH COLLATION VERSION` warning after the postgres upgrade is fixed by the two commands above (reindex included).
