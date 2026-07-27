---
title: Ditching Windows Without Losing It
date: 2026-07-27
description: Running Windows as a KVM guest instead of paying for it in the cloud
categories: [virtualization, linux]
tags: [kvm, qemu, libvirt, kubuntu, virtio, cpu-pinning]
---

I've been wanting to move off Windows as my daily driver for a while now, but a handful of applications made that harder than it sounds. Adobe Creative Cloud (InDesign, Photoshop, Illustrator), AnyDesk for remote support, and Universal Desktop for our InfoGenesis POS all only run properly on Windows. My stopgap plan had originally been to run a Windows 365 Cloud PC account — functional, but it's a recurring subscription cost for something my own laptop hardware should easily be able to do itself.

My Dell Precision 3591 has two 2TB NVMe drives and 64GB of RAM, dual-booted between Windows and Kubuntu. The plan: keep Kubuntu as the host OS, and run my existing Windows install as a KVM/QEMU virtual machine with the Windows drive attached directly as a raw block device, so I'd get near-native disk performance without a full reinstall.

> **Quick definitions, for anyone following along:**
> **KVM** (Kernel-based Virtual Machine) is Linux's built-in hypervisor. **QEMU** handles the actual hardware emulation on top of it. **libvirt** is the management layer (and `virsh`/virt-manager are how you talk to it) that most people actually interact with day to day.

## Backing up before touching anything

Before migrating a real, in-use Windows install into a VM, I wanted a full backup. This turned out to be its own small project. A raw `dd` image of the drive didn't fit on my external backup drive, and switching to `ntfsclone` ran into a second problem: exFAT doesn't support sparse files, so `ntfsclone` still wrote out the full volume size regardless. Reformatting the external drive to ext4 fixed that and let `ntfsclone` write proper partition-level images.

The other early lesson: NVMe device names aren't stable. `/dev/nvme0n1` and `/dev/nvme1n1` flipped which physical drive they pointed to between reboots more than once, which is exactly the kind of thing you want to know about *before* it silently images the wrong disk. The fix was switching everything — backup commands and, later, the VM's XML — to reference drives by their persistent `/dev/disk/by-id/` serial-based path instead.

## Getting Windows to boot as a guest

With OVMF for UEFI firmware and swtpm providing a virtual TPM 2.0 (Windows 11 checks for both), the migration needed a specific order of operations to avoid an `INACCESSIBLE_BOOT_DEVICE` failure: disable Fast Startup, disable hibernation, and suspend BitLocker on the Windows side first, since none of those survive a change in virtual hardware gracefully.

The storage bus was a two-stage process. Windows' existing install is bound to `stornvme.sys`, not the virtio-scsi driver a VM needs for good disk performance, so the first boot had to happen under SATA/AHCI emulation. Installing the virtio-win guest tools alone didn't get the driver registered as boot-capable — it took attaching a small dummy virtio disk while still booting from SATA to force Windows to properly bind the driver through Plug and Play. After that, switching the boot disk's bus from `sata` to `virtio` in the domain XML went cleanly. One XML quirk worth noting: when you change a disk's bus type, delete the old `<address>` line entirely rather than editing it — libvirt regenerates the correct address type (drive-based for SATA, PCI-based for virtio) on its own.

I hit one more boot failure later when the NVMe enumeration swap mentioned earlier caused the VM to load GRUB from the wrong disk's EFI partition instead of Windows. Same fix applied: point the VM's XML at the drive via `/dev/disk/by-id/` instead of the transient `nvme0n1`/`nvme1n1` names, and it became immune to enumeration order entirely.

## Tuning it to actually feel fast

Once Windows booted reliably, the VM was noticeably laggy under load, and it turned out to be a CPU scheduling problem rather than a lack of resources. My laptop's Core Ultra 7 165H is a hybrid chip — 6 performance cores (hyperthreaded, 12 threads) plus 10 efficiency cores (no hyperthreading) — and without CPU pinning, KVM was free to float the VM's vCPU threads across any mix of P-cores and E-cores at any moment. Windows had no idea some of its "cores" were 2-3x slower than others from one instant to the next.

The fix was pulling the actual thread-sibling pairs from `/sys/devices/system/cpu/cpu*/topology/thread_siblings_list`, then pinning the VM's 10 vCPUs to 5 dedicated P-core pairs via `<cputune>`/`<vcpupin>` in the domain XML, pinning the QEMU emulator thread to the 6th P-core pair so it never contends with a vCPU, and matching the guest's `<cpu><topology>` block (`cores='5' threads='2'`) to reflect the real hardware relationship. I also added Hyper-V enlightenments (`relaxed`, `vapic`, `spinlocks`, `synic`/`stimer`) to the domain's `<features>` block, and switched the host's CPU governor from `powersave` to `performance`.

The combination made an immediate, obvious difference — Task Manager inside the guest now correctly reports 5 cores / 10 logical processors instead of a flat, misleading view, and the VM stopped feeling like it was randomly stalling mid-task.

## Where it stands

Windows now runs as a guest under Kubuntu with virtio storage, pinned CPU cores matching my chip's real topology, and Hyper-V enlightenments enabled — close enough to bare-metal that Adobe's apps, AnyDesk, and Universal Desktop for InfoGenesis are all usable day to day. That's enough to retire the Windows 365 Cloud PC subscription and get that monthly cost back, while keeping Linux as my actual daily driver.

There's still cleanup work ahead — a couple of minor Kubuntu hardware quirks (an unresolved touchpad interrupt bug, an audio routing default) that I worked around rather than fixed — but the core goal is done: one laptop, one set of hardware, running both operating systems at effectively native speed.
