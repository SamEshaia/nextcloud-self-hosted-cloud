# Disk Expansion (300G → 600G)

This exact procedure was used to double the Nextcloud VM's disk from 300G to 600G with zero data loss. The steps are safe — `lvextend`, `growpart`, and `resize2fs` are all online, non-destructive operations — but there's one trap that cost us an outage window: **a warm reboot does not re-read the disk size.**

## The Safe Part: Three Online Operations

All three commands preserve existing data. Nothing is reformatted, deleted, or moved:

- `lvextend` adds free space from the volume group to the end of the logical volume
- `growpart` shifts the partition boundary outward to cover the new space
- `resize2fs` grows the ext4 filesystem into that space

The only risk is power loss mid-operation — LVM and ext4 are journaled, so even that recovers cleanly.

## Step 1: Extend the Logical Volume (KVM host)

Confirm the VM's disk backing first:

```bash
sudo virsh domblklist <vm_name> --details
```

```
 Type    Device   Target   Source
-------------------------------------------------------
 block   disk     vda      /dev/<vg_name>/<lv_name>
 file    cdrom    sda      -
```

Then extend:

```bash
sudo lvextend -L +300G /dev/<vg_name>/<lv_name>
```

```
  Size of logical volume <vg_name>/<lv_name> changed from 300.00 GiB (76800 extents) to 600.00 GiB (153600 extents).
  Logical volume <vg_name>/<lv_name> successfully resized.
```

## Step 2: Cold Start the VM (the gotcha)

After a plain `lvextend` + reboot, the guest still saw 300G:

```
vda    253:0    0   300G  0 disk
├─vda1 253:1    0     1M  0 part
└─vda2 253:2    0   300G  0 part /
```

And `growpart` reported:

```
NOCHANGE: partition 2 could only be grown by 2015 [fudge=2048]
```

**Why:** KVM only re-reads the block device size when the guest's disk controller is reinitialized. A warm reboot (`virsh reboot` or `sudo reboot` inside the guest) does not do this — the virtio device keeps its old capacity. You need a full power cycle.

**The wrong-looking-but-right sequence:**

```bash
sudo virsh shutdown <vm_name>
# wait for it to fully stop — check with: sudo virsh list --all
sudo virsh start <vm_name>
```

If you jump the gun and get `error: Domain is already active`, the VM was still shutting down when you tried to start it. At that point the reliable fix is a forced power-off — safe here because the filesystem is journaled and the VM is already mid-maintenance:

```bash
sudo virsh destroy <vm_name>
sudo virsh start <vm_name>
```

`destroy` is a hard power cut (like pulling the plug), not a filesystem-safe shutdown. Don't use it on a healthy VM as routine practice.

## Step 3: Grow Partition and Filesystem (inside VM)

After the cold start, SSH back in:

```bash
sudo growpart /dev/vda 2
sudo resize2fs /dev/vda2
df -h
```

Expected result:

```
/dev/vda2       589G  123G  452G  22% /
```

## Verification Checklist

- [ ] `virsh domblklist <vm_name> --details` shows the LV at the new size
- [ ] Inside the VM, `lsblk` shows `vda` at 600G (after cold start only)
- [ ] `df -h` shows the grown filesystem
- [ ] Nextcloud admin Overview (`/settings/admin/overview`) shows no storage warnings

## If It Still Won't Start

After a forced `destroy` + `start`, give the guest 60-90 seconds before the first SSH attempt — a cold boot of a 600G-disk VM with snap services starting takes longer than a warm reboot. If SSH times out, check from the host with `sudo virsh console <vm_name>` before assuming the VM is broken; a snap auto-update running at boot can delay sshd.