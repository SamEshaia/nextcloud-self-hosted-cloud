# Troubleshooting

Real failures from this deployment's history, with the actual fixes. Roughly chronological.

## tar.bz2 extraction fails: `bzip2: Cannot exec`

Ubuntu Server minimal ships without bzip2. `tar -xjvf` dies immediately.

```bash
sudo apt install -y bzip2
```

## Downloaded tarball won't extract / checksum mismatch

A truncated download (interrupted wget, network blip) produces a file that's the right name but wrong bytes — sometimes it even starts extracting before dying.

```bash
md5sum -c nextcloud-<version>.tar.bz2.md5
# if it fails: rm the tarball and re-download. Don't extract an unverified archive.
```

## MySQL backup redirect: `Permission denied`

```bash
sudo mysqldump ... > /backup/nextcloud.sql
# bash: /backup/nextcloud.sql: Permission denied
```

The shell opens the output file **as your user** before sudo elevates mysqldump. The redirect never gets sudo. Fix:

```bash
sudo mysqldump ... | sudo tee /backup/nextcloud.sql > /dev/null
```

## VM disk won't grow after lvextend

`lvextend` on the host succeeded, but inside the guest `lsblk` still shows the old size and `growpart` says `NOCHANGE`. A warm reboot does not re-read the disk size — see the full walkthrough in [Disk Expansion](disk-expansion.md). Short version: cold start required.

```bash
sudo virsh shutdown <vm_name>   # wait for it to fully stop
sudo virsh start <vm_name>
```

## `virsh start` says "Domain is already active"

The VM was still shutting down when the start command ran. If it's wedged mid-shutdown:

```bash
sudo virsh destroy <vm_name>    # forced power-off — safe for planned maintenance on a journaled fs
sudo virsh start <vm_name>
```

Don't make `destroy` a habit on healthy VMs — it's equivalent to pulling the power cord.

## VM unreachable after forced power cycle

After `destroy` + `start`, SSH timed out for a while. Before assuming the VM is dead:

- Wait 60-90 seconds — cold boot with snap/apt services starting is slower than a warm reboot
- Check from the host: `sudo virsh list --all` (state) and `sudo virsh console <vm_name>` (boot messages)
- If you typed the IP by hand, re-check it — a transposed octet produces the exact same "connection timed out" symptom (this one got us)

## NAT-forwarded VM is unreachable from some VLANs

The VM originally sat behind a NAT + iptables-forwarded libvirt network. Reachability was inconsistent depending on source VLAN and return-path rules.

**Fix:** switch the guest NIC to **bridged** networking so the VM gets its own IP on the lab VLAN. Bridging removes the whole class of iptables-forwarding and hairpin-NAT problems. (Virt-manager: NIC → Network source: Bridge device.)

## Expired Let's Encrypt certificate

`certbot renew` failing silently until the cert actually expires. Diagnose:

```bash
sudo certbot renew --dry-run
sudo tail -50 /var/log/letsencrypt/letsencrypt.log
```

Common causes, in order of likelihood: port 80 no longer reachable from the internet (edge forwarding changed), the domain's DNS record no longer points at this edge, or Apache config errors (`apache2ctl configtest`). Renewal only works if the whole chain — DNS → edge → VM:80 → ACME path — is intact.

## Nextcloud background jobs not running

Files stop syncing changes, activity feed goes stale, but the UI works. Check whether cron/webcron/systemd-timer is actually configured:

```bash
sudo -u www-data php /var/www/nextcloud/occ background:cron
sudo -u www-data php /var/www/nextcloud/occ config:app:get core backgroundjobs_mode
```

Set the mode explicitly and install the crontab entry (see [Installation step 10](installation.md)).

## WebDAV login works in browser but clients fail

Nextcloud's bruteforce protection throttles repeated failed logins — a mistyped password in a phone client can lock the account's IP out for a while even after the password is fixed. Check and clear:

```bash
sudo -u www-data php /var/www/nextcloud/occ security:bruteforce:reset <client_ip>
```

## Apache SSL/TLS warnings in Admin → Overview

Missing `ServerName`, mismatched vhost names, or missing headers (`Strict-Transport-Security`, `X-Content-Type-Options`) all show up there. The vhost template in [configs/](../configs/apache-vhost.conf) includes the header block; run `sudo apache2ctl configtest` after any vhost edit.