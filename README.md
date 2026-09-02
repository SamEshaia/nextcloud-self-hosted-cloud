# Nextcloud Self-Hosted Cloud

Self-hosted Nextcloud running as a KVM virtual machine behind Cloudflare — migrated off a snap install onto a dedicated tarball deployment with MySQL, Let's Encrypt TLS, VM disk expansion, and real-world troubleshooting baked in.

## What This Is

This repo documents a real, production Nextcloud deployment: an Ubuntu VM under KVM/libvirt on a VLAN-segmented home network, serving a public cloud endpoint through Cloudflare with Let's Encrypt TLS. It covers the parts most guides skip — migrating off a snap install to a tarball deployment, resizing the VM's disk after LVM runs out (including the cold-start gotcha), and what's actually exposed to the internet versus what stays behind the firewall.

## Architecture

```
Internet → Cloudflare (TLS termination, CDN)
               ↓ origin pull over HTTPS
         <nextcloud_hostname>  (public DNS → Cloudflare edge)
               ↓
   ┌─────────────────────────────────────────────┐
   │  KVM host (<kvm_host_name>, VLAN 2)         │
   │  └─ VM: <vm_name>  — Ubuntu 24.04           │
   │     600G system disk (RAID6 LVM backing)    │
   │     Nextcloud tarball (Apache + PHP + MySQL)│
   │     Let's Encrypt cert via certbot          │
   └─────────────────────────────────────────────┘
               ↓
        LAN clients (VLAN-segmented)
        Remote clients → Tailscale (admin access)
```

## Key Features

- **KVM/libvirt VM** — Nextcloud isolated from the LAN and from the KVM host itself
- **Tarball deployment** — deliberately migrated off the Nextcloud snap for direct control of Apache, PHP, and MySQL
- **600GB system disk** — expanded from 300GB via LVM `lvextend` + `growpart` + `resize2fs` (docs include the cold-start gotcha)
- **Cloudflare front end** — public endpoint without exposing router ports; TLS terminated at the edge
- **Let's Encrypt automation** — certbot inside the VM with renewal timer
- **VLAN segmentation** — the VM lives on its own lab VLAN, separate from main LAN
- **Apps**: Talk (video/chat), Calendar, Cospend

## Documentation

- [Architecture](docs/architecture.md) — network path, ports, TLS chain, what's exposed
- [Installation](docs/installation.md) — host prep, VM creation, Nextcloud snap setup
- [Disk Expansion](docs/disk-expansion.md) — 300G → 600G with the warm-reboot trap and fix
- [Maintenance](docs/maintenance.md) — snap upgrades, backups, cron, health checks
- [Security Notes](docs/security.md) — hardening checklist, what this deployment gets right and wrong
- [Troubleshooting](docs/troubleshooting.md) — real failures hit during setup and how they were fixed

## Config Examples

All config files in `configs/` are sanitized templates with placeholder values. Replace `<placeholder>` text with your actual values before use.

## Hardware Used

- **KVM host:** <server_hardware> — RAID6 LVM volume group backing VM disks
- **VM:** 4 vCPU / 8GB RAM class guest, Ubuntu Server 24.04 LTS
- **Storage:** 600G virtio disk (grew from 300G), backed by RAID6 + LVM
- **Network:** pfSense router with VLAN segmentation, Tailscale for remote admin

## Requirements

- Ubuntu Server 24.04 LTS (or 22.04+) for the VM
- KVM/libvirt host with a volume group that has headroom for the VM disk
- A domain on Cloudflare (any registrar works — Cloudflare just needs DNS)
- Port 80/443 reachable from the internet at your edge (port-forward or tunnel — see Architecture)

## License

MIT — see [LICENSE](LICENSE). This is documentation and configuration, not Nextcloud software itself.