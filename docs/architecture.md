# Architecture

## Network Path

Public traffic terminates at Cloudflare and is pulled from the VM over HTTPS. Internal clients can bypass the edge and hit the VM directly on the lab VLAN.

```
                    ┌──────────────┐
   Public users ───▶│  Cloudflare  │  TLS edge, CDN, brute-force shielding
                    └──────┬───────┘
                           │ HTTPS origin pull (443)
                           ▼
              <nextcloud_hostname>  (public DNS record → Cloudflare edge)
                           │
                           ▼  routed to VM:443
   ┌───────────────────────────────────────────────────┐
   │ KVM host  <kvm_host_name>   — lab VLAN (<lab_vlan>)│
   │                                                    │
   │  virsh domain: <vm_name>                           │
   │  ├─ virtio disk vda: 600G (LV on RAID6 VG)         │
   │  ├─ VM IP: <vm_ip> on <lab_vlan>                   │
   │  ├─ Nextcloud tarball under /var/www (Apache :80/:443,│
   │  │   PHP-FPM, MySQL)                               │
   │  └─ certbot (Let's Encrypt renewal timer)          │
   └───────────────────────────────────────────────────┘
```

## Ports

| Port | Service | Bound | Exposure |
|------|---------|-------|----------|
| 80   | Apache (redirect + ACME challenge) | VM | needed at edge for cert issuance/renewal |
| 443  | Apache TLS (Nextcloud) | VM | needed at edge; Cloudflare origin pull |
| 3306 | MySQL | localhost only | never exposed |
| 9090 | Cockpit | VM | LAN/Tailscale only — never published |
| 22   | SSH | VM | LAN/Tailscale only |

The edge (router or tunnel) forwards 80/443 to the VM. Nothing else is published.

## TLS Chain

- Browser ↔ Cloudflare: Cloudflare edge certificate
- Cloudflare ↔ VM: Let's Encrypt certificate issued for `<nextcloud_hostname>`, managed by certbot inside the VM
- The VM's Apache redirects all HTTP → HTTPS; the ACME HTTP-01 challenge is served on port 80

## What's Exposed vs. Internal

**Public internet sees:**
- The Nextcloud login page and WebDAV endpoints on `<nextcloud_hostname>`
- Nextcloud version (via `/status.php`) — standard for Nextcloud unless suppressed

**Stays internal (lab VLAN + Tailscale only):**
- SSH into the KVM host and VM
- Cockpit (`:9090`) on the VM for web admin
- MySQL, snap internals, backups

## Failure Domains

| Layer | Failure | Recovery |
|-------|---------|----------|
| Cloudflare | Edge outage | Origin still serves LAN clients directly |
| VM | Guest hang/crash | `virsh destroy` + `start` from host (see Troubleshooting) |
| Storage | LV out of space | Expand: see [Disk Expansion](disk-expansion.md) |
| Cert | Renewal failure | Re-run certbot with `--force-renewal`; needs port 80 reachable |

## Capacity Notes

- 600G virtual disk on a RAID6 (2.2TB) volume group shared with other VMs — expansion procedure documented for when headroom runs low
- MySQL + PHP + Apache on the same VM — RAM budget ~1-2GB under normal use
- Nextcloud data directory lives under `/var/www/<nextcloud_docroot>/data` — sized against the 600G disk