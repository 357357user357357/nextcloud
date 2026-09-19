# nextcloud

State + restore docs for the **Nextcloud stack on the RU relay server** (plain HTTP by raw IP, co-hosted with the xray VPN relay — see the `vpn` repo for that side).

Server: Ubuntu **26.04.1**, kernel 7.0.0-31-generic. Stack: apache2 + php8.5-fpm + postgresql + redis + APCu.

| Component | Version |
|---|---|
| Nextcloud | **33.0.9** (2026-09-19: 32.0.3.2 → 32.0.15 → 33.0.9) |
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

## The PHP 8.5 gate patch (NC 32 only — RETIRED since NC 33)

Historical (NC 32 era): Ubuntu 26.04 ships **only PHP 8.5**, while Nextcloud 32 hard-caps `PHP_VERSION_ID >= 80500` → refuses to run. Patch (backup kept at `/root/versioncheck.php.bak`):

```diff
--- a/lib/versioncheck.php
+++ b/lib/versioncheck.php
@@ -17 +17 @@
-if (PHP_VERSION_ID >= 80500) {
+if (PHP_VERSION_ID >= 80600) {
```

Apply: `sed -i 's/PHP_VERSION_ID >= 80500/PHP_VERSION_ID >= 80600/' /var/www/nextcloud/lib/versioncheck.php`

✅ **Retired 2026-09-19**: Nextcloud **33** natively supports PHP 8.5 (its `versioncheck.php` rejects only `>= 80600`) — no patch needed on 33+. Keep the sed recipe for any restore onto a 32.x webroot (step 6 of the guide below).

## Post-upgrade DB maintenance (26.04 → newer postgres collation)

```bash
sudo -u postgres psql -d nextcloud -c 'REFRESH COLLATION VERSION;'
sudo -u postgres reindexdb -d nextcloud
```

## Restore / rebuild guide

1. Ubuntu 26.04 + packages: `apache2 libapache2-mod-php8.5 php8.5-{fpm,pgsql,curl,gd,xml,mbstring,zip,intl,apcu} postgresql redis-server php-redis`.
2. Copy this repo's apache/php files into place; `a2enmod php8.5 rewrite headers env dir mime setenvif; a2ensite nextcloud; a2dissite 000-default` (keep 8443 ssl default if desired).
3. Drop the Nextcloud webroot + `data/` from backup (or install **33.0.9** — use the GitHub source tag tarball, it includes `apps/`; restore `data/` + `config/config.php` from `config.php.example` with real secrets).
4. `chown -R www-data:www-data /var/www/nextcloud`.
5. Restore postgres: `sudo -u postgres pg_restore -d nextcloud <dump>` (or full cluster `pg_restore`/`pg_upgrade` from the old host).
6. versioncheck patch **only for a 32.x webroot** (33+ needs nothing). Check `curl http://<RELAY_IP>/status.php` → `"installed":true,"maintenance":false`.
7. `php occ db:add-missing-indices` if status complains; REFRESH COLLATION VERSION + reindex per above.

## NC 33 upgrade lessons (2026-09-19: 32.0.3.2 → 32.0.15 → 33.0.9)

- **The updater only offers the next major when you're on the latest point release of the current major.** 32.0.3.2 saw only "32.0.15 available"; 33 unlocked after the 32.0.15 hop. Two-hop path, both done with tarball swaps (`occ upgrade` after each).
- **The download.nextcloud.com NC 33 tarball ships NO `apps/`** (and a stub `config/`). Shipped apps came from the GitHub source tag (`codeload.github.com/nextcloud/server/tar.gz/refs/tags/v33.0.9`); user apps (contacts, calendar) were copied over from the old tree and bumped with `occ app:update contacts` (8.1.2 → 8.9.0).
- **Manual-swap nesting trap**: older tarballs ship `apps/`, NC 33 ships a stub `config/` — `mv old/config /var/www/nextcloud/config` **nests** (`config/config/config.php`) instead of replacing. occ then reports "Nextcloud is not installed" and creates a 0-byte `config.php`. Always copy files explicitly: `cp -a old/config/config.php new/config/` and `rm -f new/config/CAN_INSTALL`.
- **`occ upgrade` can abort on app-store API hiccups** mid-run ("no space"/exception traces). Run `occ config:system:set appstoreenabled --type boolean --value false` before `occ upgrade`, re-enable after; do `occ app:update --all` **outside maintenance mode** (in maintenance it only lists updates).
- **Staging peak needs ~2.5 GB free** (zip + extracted tree + retained old tree on a 15 GB disk). `apt-get autoremove` + `apt-get clean` first; delete each staging dir between hops; keep the previous webroot as rollback until verified.
- **NC 31+ uses `.ncdata`** (not `.ocdata`) as the data-dir marker — don't panic when `.ocdata` is missing.
- Verify data survived: `sudo -u postgres psql nextcloud -tc 'select count(*) from oc_cards'` — 1739 cards / 2 addressbooks before AND after.
- Final state: `status.php` → `{"installed":true,"maintenance":false,"needsDbUpgrade":false,"version":"33.0.9.1"}`; contacts 8.9.0 enabled; calendar functional.

## Lessons from the 26.04 upgrade (2026-09)

- **Check disk first** — the dist-upgrade stalled with 0 bytes free; a hung `redis-server.postinst` held the dpkg lock for 8.5h (`dpkg --audit`, `fuser -v /var/lib/dpkg/lock-frontend` to find it).
- Noble leftovers in sources.list break the 26.04 apt run — purge before full-upgrade.
- The upgrade removes php8.3-fpm and installs nothing in its place → Nextcloud down until the php8.5 stack is installed + vhost/socket paths updated.
- `REFRESH COLLATION VERSION` warning after the postgres upgrade is fixed by the two commands above (reindex included).
