# Maintenance

## Nextcloud Upgrades (tarball)

The tarball deployment upgrades with the official `occ` upgrade process — no snap refreshes here:

```bash
# 0. Back up first (see below)
sudo -u www-data php /var/www/nextcloud/occ maintenance:mode --on

# 1. Back up current install
sudo rsync -a /var/www/nextcloud/ /backup/nextcloud-$(date +%F)/
sudo mysqldump --single-transaction --default-character-set=utf8mb4 \
  -u nextcloud -p nextcloud | sudo tee /backup/nextcloud-db-$(date +%F).sql > /dev/null

# 2. Download + verify new release
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-<new_version>.tar.bz2
wget https://download.nextcloud.com/server/releases/nextcloud-<new_version>.tar.bz2.md5
md5sum -c nextcloud-<new_version>.tar.bz2.md5

# 3. Swap code, keep data + config
sudo -u www-data php /var/www/nextcloud/updater/updater.phar
# or the manual path: move old, extract new, copy config/ and data/ back

# 4. Finish
sudo -u www-data php /var/www/nextcloud/occ upgrade
sudo -u www-data php /var/www/nextcloud/occ maintenance:mode --off
```

Verify version after: `/status.php` should report the new version string, and Administration → Overview should be clean.

## MySQL Maintenance

- Backups: `mysqldump --single-transaction` (safe on a live DB — no locks for InnoDB)
- Check DB size growth occasionally: tables like `oc_filecache` grow with the file count
- If the DB was dumped/restored during migration, run `occ maintenance:repair` once and re-check Admin → Overview for missing index warnings

## Certbot / TLS

```bash
sudo certbot renew --dry-run        # prove renewal works before you need it
sudo systemctl list-timers | grep -E 'certbot|letsencrypt'
```

If renewal fails, the usual causes: port 80 blocked at the edge, DNS record pointing elsewhere, or Apache configtest failing. Fix the cause — don't just force-renew.

## Disk Headroom

The VM has a 600G disk; watch `df -h` inside the guest. When free space gets low:

1. Clean Nextcloud trash + versions: `occ trash:purge` / check retention in admin settings
2. Expand the LV + partition + filesystem: [Disk Expansion](disk-expansion.md)

## Backups (what exists, what's still worth adding)

**In place:** manual mysqldump + rsync of the data directory before upgrades.

**Recommended additions (not yet automated):**
- Nightly automated dump to a second host or external mount
- restic/borg against the data directory with a remote target
- A restore test — an untested backup is a hope, not a backup

## Health Checks

```bash
# From anywhere: is it up, what version, maintenance mode?
curl -s https://<nextcloud_hostname>/status.php | python3 -m json.tool

# Inside the VM: service state
systemctl status apache2 mariadb php*-fpm 2>/dev/null | grep -E '●|Active'

# occ sanity
sudo -u www-data php /var/www/nextcloud/occ status
sudo -u www-data php /var/www/nextcloud/occ check
```

## Log Locations

| What | Where |
|------|-------|
| Nextcloud app log | `/var/www/<docroot>/data/nextcloud.log` (or Admin → Logging) |
| Apache | `/var/log/apache2/` |
| certbot | `/var/log/letsencrypt/letsencrypt.log` |
| MySQL | `/var/log/mysql/` |