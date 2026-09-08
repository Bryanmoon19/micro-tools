---
layout: post.njk
title: "Leaving VMware Just Got Harder — The 2026 Proxmox Migration Playbook"
date: 2026-09-08
description: "Broadcom keeps making VMware harder to leave and harder to stay — the latest move pulled VDDK downloads behind a support wall. If you're done with the licensing squeeze, here's the step-by-step path off ESXi onto Proxmox VE, including the gotchas (VDDK, vCenter dependencies, backup tooling) that trip people up."
tags: ["vmware", "esxi", "proxmox", "proxmox-ve", "broadcom", "migration", "virtualization", "hypervisor", "homelab", "self-hosted", "vddk", "vcenter", "qcow2", "vmdk"]
author: "Bryan Moon"
canonical: "https://devhandbook.io/blog/2026-09-08-vmware-to-proxmox-migration-playbook"
affiliate: true
cta: true
---

If you've been running VMware ESXi in a homelab or a small business, the last two years have felt like a slow-motion breakup. Broadcom's acquisition of VMware brought license changes, support gutting, and a steady stream of "wait, they did *what*?" moments. The latest: **Broadcom pulled VDDK (Virtual Disk Development Kit) downloads behind a support-contract wall**, which broke a whole ecosystem of backup and migration tools that depended on it.

The HN thread hit 151 points. The autocomplete tells the rest of the story: people are searching "vmware to proxmox migration," "esxi alternative 2026," "broadcom vmware alternative."

Here's the thing nobody's writing clearly: **leaving VMware is now harder than it used to be, but it's still very doable — and staying is getting more expensive every quarter.** This post is the exit plan. Not a rehash of Proxmox basics — a de-risking guide for the actual migration, including the gotchas that trip people up.

## Why People Are Leaving (and Why It's Getting Harder)

Let me be precise about what's actually changed, because there's a lot of panic and a lot of misinformation.

**The licensing squeeze.** Broadcom killed perpetual licenses and moved VMware to a subscription model. For homelabbers, the free ESXi license was quietly sunset. For SMBs, the per-core pricing restructure meant bills that went up 2-3× overnight. The "VMware is free for home use" era is over.

**The support gutting.** Broadcom laid off large portions of the VMware team and consolidated support. Long-time admins report ticket times that went from hours to days. The community forums — once the lifeblood of VMware troubleshooting — were gutted and moved.

**The VDDK pull (the new one).** The Virtual Disk Development Kit is the library that lets third-party tools read and write VMware disk images. Backup products (Veeam, Nakivo, and a dozen others), migration tools, and forensic utilities all depend on it. Broadcom moved VDDK downloads behind a support-contract login, which means:

- Open-source and free tools that bundled VDDK can no longer legally redistribute it
- Backup vendors had to scramble to renegotiate or re-architect
- Homelabbers who relied on free VDDK-based tooling lost access overnight

The irony is thick: **the tooling you'd use to *leave* VMware is now harder to get.** That's the "harder to leave" part. But it's not a wall — it's a speed bump, and this post walks you around it.

## The Migration Path, Step by Step

The good news: migrating from ESXi to Proxmox VE is a well-trodden path, and Proxmox has first-class tooling for it. Here's the sequence I recommend, in order of least risk.

### Step 0: Inventory Before You Touch Anything

Before you migrate a single VM, document what you actually have. This is the step everyone skips and everyone regrets skipping.

- **List every VM** — name, OS, vCPU, RAM, disk sizes, and which datastore it lives on
- **Note the disk controller type** — this matters more than you think (see the gotchas below). ESXi VMs are usually SCSI (`lsilogic` or `pvscsi`); Proxmox defaults to SATA or VirtIO SCSI
- **Note the network config** — static IPs, VLAN tags, MAC addresses (if anything depends on them)
- **Identify dependencies** — vCenter features you actually use (vMotion, DRS, templates, distributed switches). Most homelabbers use almost none of these, which makes the migration *easier*, not harder.

### Step 1: Export the VM from ESXi

There are two ways to get a VM off ESXi, and which one you use depends on whether you still have working access to the ESXi host.

**Option A — OVF export (preferred, when it works).** From the ESXi web UI or `ovftool`, export the VM as an OVF/OVA. This bundles the VM config and disks into a portable format. Proxmox can import OVF directly:

```bash
# On the Proxmox host
qm importovf <vmid> <vm.ovf> <storage>
```

**Option B — raw disk copy (when OVF export fails).** If OVF export is broken (and with the VDDK mess, some tooling is), you can copy the raw VMDK directly. Shut the VM down, then from the ESXi host:

```bash
# On ESXi, find the VMDK path
ls /vmfs/volumes/<datastore>/<vm>/

# Copy it off (scp to your Proxmox host or a staging box)
scp /vmfs/volumes/<datastore>/<vm>/<vm>.vmdk user@proxmox:/tmp/
```

The catch: a VMDK is often split into multiple files (`-flat.vmdk`, `-delta.vmdk`, etc.) or is a thin-provisioned descriptor pointing at a `-flat` file. You need the *flat* file (the actual data), not just the small descriptor. If you only copy the descriptor, you'll get a tiny file and a broken VM.

### Step 2: Convert the Disk

Proxmox uses `qcow2` (or raw) disk images. You need to convert the VMDK. The tool is `qemu-img`, which ships with Proxmox:

```bash
# Convert VMDK to qcow2
qemu-img convert -f vmdk -O qcow2 /tmp/<vm>.vmdk /tmp/<vm>.qcow2
```

If the VMDK is split across multiple files, you can point `qemu-img` at the *descriptor* VMDK and it'll follow the chain — but only if all the pieces are in the same directory.

### Step 3: Create the VM in Proxmox and Attach the Disk

Create a new VM in Proxmox with matching specs (vCPU, RAM), then import the converted disk:

```bash
# Create the VM (no disk yet)
qm create <vmid> --name <vm-name> --memory 4096 --cores 4 --net0 virtio,bridge=vmbr0

# Import the converted disk
qm importdisk <vmid> /tmp/<vm>.qcow2 <storage>

# Attach it as the boot disk
qm set <vmid> --scsihw virtio-scsi-pci --scsi0 <storage>:vm-<vmid>-disk-0
```

### Step 4: Fix the Boot and Drivers

This is where most migrations go sideways. The VM boots, but it blue-screens, kernel-panics, or drops to a recovery shell. The cause is almost always the same: **the disk controller changed.**

ESXi VMs use VMware's SCSI controllers (`lsilogic` or `pvscsi`). Proxmox defaults to VirtIO SCSI. When the OS boots and can't find its root disk on the new controller, it panics.

The fix depends on the guest OS:

- **Linux:** VirtIO drivers are in the kernel for most modern distros, but the *initramfs* may not include them. Boot a live CD, chroot in, and rebuild initramfs with `virtio_scsi` and `virtio_blk` included. Or, simpler: set the disk to SATA (`--scsi0` → `--sata0`) for the first boot, install the VirtIO drivers, then switch back.
- **Windows:** Install the VirtIO drivers *before* you migrate (or attach the VirtIO driver ISO and load them during first boot). Windows will otherwise BSOD with `INACCESSIBLE_BOOT_DEVICE`.

The pragmatic shortcut: **use SATA for the disk controller on first boot.** It's slower than VirtIO SCSI, but it's universally supported and lets the VM boot so you can install the right drivers, then switch to VirtIO SCSI for performance.

### Step 5: Recreate Networking

Proxmox uses Linux bridges (`vmbr0`, `vmbr1`, etc.) instead of VMware's vSwitches. Map your old port groups to bridges:

- Default VM network → `vmbr0` (bridged to your physical NIC)
- VLAN-tagged networks → create a bridge with a VLAN tag, or use VLAN-aware bridges

If anything depends on a specific MAC address (DHCP reservations, license servers), set it explicitly:

```bash
qm set <vmid> --net0 virtio=<MAC>,bridge=vmbr0
```

## The Gotchas (the part generic guides skip)

These are the things that actually bite people, in order of how often I see them:

1. **The disk controller change is the #1 failure.** Plan for it. Boot with SATA first, install VirtIO drivers, then switch. Don't try to go straight to VirtIO SCSI on a migrated VM.

2. **VDDK-based backup tools break.** If you were using a free backup tool that bundled VDDK, it may stop working after the pull. The migration itself doesn't need VDDK (you're using `qemu-img` and `qm importovf`), but your *old* backup chain does. Export your backups before you decommission the ESXi host.

3. **vCenter dependencies.** If you used vCenter for templates, clones, or distributed switches, none of that maps 1:1 to Proxmox. Templates become Proxmox templates (via `qm template`), clones become `qm clone`, and distributed switches become VLAN-aware bridges. It's all doable, but it's manual — budget time for it.

4. **Thin vs. thick provisioning.** A thin-provisioned VMDK converted to qcow2 stays thin (good), but if you convert to raw, it becomes thick and eats the full allocated size. Use qcow2 unless you have a reason not to.

5. **Snapshots don't migrate.** VMware snapshots (the `-delta.vmdk` files) don't survive the conversion cleanly. Consolidate/delete snapshots on the ESXi side *before* you export, or you'll be migrating a half-merged disk chain.

6. **The free ESXi license is gone.** If you're still on a free ESXi license, you can't even get VDDK or support anymore. That's the clearest signal that the exit is now, not later.

## What You Actually Lose (the honest tradeoffs)

Proxmox isn't a drop-in VMware replacement, and pretending it is sets you up for disappointment. Here's the honest list:

- **vMotion/DRS** — Proxmox has live migration (`qm migrate`), but it's not as polished as vMotion, and there's no DRS equivalent for automatic load balancing. For a homelab or small SMB, you won't miss it.
- **vCenter's single pane of glass** — Proxmox's web UI is per-node (or per-cluster with the cluster view). It's good, but it's not vCenter. If you have 3+ hosts, the cluster view covers most of what you need.
- **Third-party ecosystem** — VMware has decades of third-party tooling. Proxmox's ecosystem is smaller but growing, and the core (backup via `vzdump`, replication, HA) is built in and free.

What you *gain*: no licensing, no per-core pricing, no support-contract wall, and a hypervisor that's actively developed in the open. For most homelabbers and SMBs, that trade is a no-brainer.

## The Bottom Line

Broadcom is making VMware harder to leave — but the exit is still a weekend project, not a career change. Export your VMs, convert the disks with `qemu-img`, boot with SATA first, install VirtIO drivers, and you're off. The VDDK pull is annoying, but it doesn't block the migration path — it just means you should export your backups *before* you pull the plug on ESXi.

If you've been waiting for a sign, this is it. The free ESXi license is gone, the pricing only goes one direction, and the tooling to leave is getting harder to reach. Migrate now, while the path is still well-trodden.

---

*Running Proxmox already? Check out our guides on [Portainer alternatives for Proxmox LXC](/blog/portainer-alternatives-proxmox-lxc-2026/), [TrueNAS migration](/blog/truenas-core-eol-migration-path/), and [running Home Assistant in an LXC](/blog/proxmox-home-assistant-lxc/).*
