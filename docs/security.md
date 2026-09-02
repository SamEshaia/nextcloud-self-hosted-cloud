# Security Notes

An honest read of this deployment — what's solid, and what's knowingly imperfect. Written from a Sec+ study perspective where "identify the gap" is half the exam.

## What's Solid

- **Edge TLS everywhere** — HSTS from Cloudflare, HTTP → HTTPS redirect at the VM, Let's Encrypt with an automated renewal timer
- **VLAN segmentation** — the VM sits on a dedicated lab VLAN, not flat on the main LAN; lateral movement from a compromised guest is slowed
- **Management planes unpublished** — Cockpit (:9090) and SSH are LAN/Tailscale-only; only 80/443 are reachable from the internet
- **Database loopback-only** — MySQL binds localhost; no network-exposed DB
- **Checksum-verified releases** — tarballs are md5-verified before extraction
- **Brute-force protection** — Nextcloud's built-in bruteforce throttling is active on login/WebDAV endpoints (visible in `/capabilities` as the `bruteforce` capability)

## Known Gaps (documented, not hidden)

1. **No zero-trust gate in front of the login page.** Other services in this homelab sit behind Cloudflare Access; the Nextcloud hostname does not — anyone on the internet reaches the Nextcloud login directly. This is deliberate (WebDAV + mobile clients want native auth), but it makes the login endpoint the attack surface. Nextcloud's built-in bruteforce protection + strong passwords carry that weight. A Cloudflare Access self-hosted app policy on `/login` (with service-auth bypasses for WebDAV paths) is the next hardening step if needed.

2. **Version disclosure.** `/status.php` reports the exact version (e.g. 33.0.6). Standard Nextcloud behavior; attackers use it for CVE matching. Mitigation is boring and effective: patch promptly (minor upgrades take minutes — see [Maintenance](maintenance.md)).

3. **Single-VM blast radius.** Web, DB, and data share one guest. A VM escape or full compromise takes everything. Backups to a *different* host are the honest mitigation — see Maintenance.

4. **Backup restore untested.** Manual dumps exist; an automated, restore-tested pipeline doesn't yet. Highest-value next investment.

5. **fail2ban not present.** Nextcloud bruteforce throttling covers the app layer, but SSH still relies on key-only auth and Cloudflare/edge rate limiting. fail2ban with the Nextcloud log filter would add a network-layer hammer.

## Hardening Checklist (applied)

- [x] SSH key-only auth (password auth disabled on lab hosts)
- [x] Only 80/443 published; everything else LAN/Tailscale
- [x] MySQL bound to localhost
- [x] Release checksums verified before deploy
- [x] Automated cert renewal timer active
- [x] Nextcloud security scan (scan.nextcloud.com) — keep it at grade A

## Hardening Checklist (planned)

- [ ] Automated off-host backups + documented restore test
- [ ] fail2ban with nextcloud filter
- [ ] Consider Cloudflare Access policy on `/login` for admin accounts
- [ ] Monthly `apt list --upgradable` review inside the VM (unattended-upgrades for security only)

## Threat Model Notes

- **Who's the adversary?** Opportunistic scanners (constant), credential stuffers (WebDAV + login endpoints get hammered), not targeted APTs. Defenses are sized accordingly.
- **Data classification:** personal files, documents, bug-bounty write-ups. Not payment data, not regulated health data. Loss would be annoying, not existential — but the bug-bounty reports folder makes the backup story more than academic.
- **Recovery posture:** if the VM dies, the data directory + DB dump rebuilds the instance on a fresh guest in under an hour using [Installation](installation.md).