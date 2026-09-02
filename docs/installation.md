# Installation

The deployment went through two phases: an original snap install (retired), then a deliberate migration to a tarball deployment inside a dedicated KVM VM. The tarball version is what's documented here — it gives direct control over Apache, PHP, and MySQL instead of going through snap wrappers.

## Phase 0: The Snap Era (what we left behind)

The first Nextcloud ran as the official snap on a different server. It worked, but the snap's opinionated wrappers made Apache/PHP tweaks awkward and the instance shared a box with unrelated services. The migration path was: build a clean VM, install MySQL and Apache by hand, dump the old database, rsync the data directory, and cut over.

## 1. Create the VM

On the KVM host (virt-manager or virsh both work):

- Ubuntu Server 24.04 LTS guest
- virtio disk (starts at 300G — sized up later, see [Disk Expansion](disk-expansion.md))
- Network: **bridged** onto the lab VLAN so the VM gets a routable LAN IP (NAT + iptables forwarding also works, but bridging avoids a class of connectivity bugs — see [Troubleshooting](troubleshooting.md))
- 4 vCPU / 8GB class guest

```bash
sudo virsh list --all          # confirm the domain exists
sudo virsh start <vm_name>
sudo virsh domblklist <vm_name> --details
```

## 2. Base Server Prep (inside VM)

Ubuntu Server minimal images ship without some basics — hit these before they bite:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y nano bzip2 curl          # bzip2 missing = tar.bz2 extraction fails
timedatectl set-timezone <timezone>
```

## 3. Web Stack

```bash
sudo apt install -y apache2 mariadb-server php php-mysql php-xml php-mbstring \
  php-gd php-curl php-zip php-intl php-apcu libapache2-mod-php
```

Enable required Apache modules and harden PHP per the [Nextcloud admin manual](https://docs.nextcloud.com/server/stable/admin_manual/installation/source_installation.html):

```bash
sudo a2enmod rewrite headers env dir mime ssl
```

## 4. Verify the Tarball (don't skip this)

Download the release and its checksum from nextcloud.com, then verify **before** extracting:

```bash
cd ~/Downloads
wget https://download.nextcloud.com/server/releases/nextcloud-<version>.tar.bz2
wget https://download.nextcloud.com/server/releases/nextcloud-<version>.tar.bz2.md5
md5sum -c nextcloud-<version>.tar.bz2.md5
```

Gotcha: a partially-downloaded file passes `tar` until it doesn't — the checksum step is what catches it. And if `tar` fails with `bzip2: Cannot exec`, you skipped step 2.

## 5. Deploy

```bash
sudo tar -xjvf nextcloud-<version>.tar.bz2 -C /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
```

## 6. Database

```bash
sudo mysql -e "CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
sudo mysql -e "CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY '<db_password>';"
sudo mysql -e "GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"
```

## 7. Migrating From an Existing Install (optional)

This is how the snap instance's data moved over. Dump from the old box first — and note the classic redirect gotcha:

```bash
# FAILS — the shell creates the file as your user before sudo kicks in:
sudo mysqldump --single-transaction --default-character-set=utf8mb4 \
  -h localhost -u nextcloud -p nextcloud > /some/backup/nextcloud.sql
# bash: nextcloud.sql: Permission denied

# WORKS — write with sudo, or pipe through tee:
sudo mysqldump --single-transaction --default-character-set=utf8mb4 \
  -h localhost -u nextcloud -p nextcloud | sudo tee /backup/nextcloud.sql > /dev/null
```

Then on the new VM:

```bash
sudo mysql nextcloud < nextcloud.sql
sudo rsync -av /old/nextcloud/data/ /var/www/nextcloud/data/
sudo cp /old/nextcloud/config/config.php /var/www/nextcloud/config/
sudo chown -R www-data:www-data /var/www/nextcloud
```

Edit the copied `config.php`: update `dbname`/`dbuser`/`dbpassword` if they changed, verify `datadirectory`, and confirm `trusted_domains` includes the public hostname.

## 8. Apache VirtualHost

`/etc/apache2/sites-available/nextcloud.conf` (template in [configs/](../configs/apache-vhost.conf)):

```apache
<VirtualHost *:80>
    ServerName <nextcloud_hostname>
    DocumentRoot /var/www/nextcloud/

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>
</VirtualHost>
```

```bash
sudo a2ensite nextcloud
sudo a2dissite 000-default
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Set `ServerName` explicitly — mismatched names cause the classic warnings in Nextcloud's admin overview, and certificate issuance needs the name to resolve to this server.

## 9. Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d <nextcloud_hostname>
sudo systemctl list-timers | grep certbot    # confirm the renewal timer exists
```

Renewal requires port 80 to stay reachable from the internet at the edge — see [Architecture](architecture.md).

## 10. Background Jobs

Nextcloud needs a cron-style background job. Either system cron for `www-data`:

```bash
sudo -u www-data crontab -e
# */5 * * * * php -f /var/www/nextcloud/cron.php
```

Or a systemd timer. Verify at Administration → Basic settings → Background jobs (web cron shows as a working option only if explicitly enabled).

## 11. First Login

Complete the installer at `https://<nextcloud_hostname>` (or finish config.php manually if migrating), then check Administration → Overview for warnings: missing indexes, memory cache, phone region. Fix them now — the overview page is the cheapest health check the platform gives you.