+++
title = 'Raspberry Pi Boot Recovery: fstab mount failures and forgotten init= in cmdline.txt'
date = 2026-10-05T02:59:03+08:00
draft = false
tags = ["raspberry-pi", "linux", "troubleshooting", "fstab", "boot"]
+++

I recently hit a Raspberry Pi boot failure that looked random at first, but it turned out to be two stacked issues:

1. stale external-drive mount entries in `/etc/fstab`
2. a leftover recovery parameter (`init=/bin/sh`) in `cmdline.txt`

This post is a clean recovery playbook from that incident.

## Symptoms I observed

- boot error related to root/emergency shell (for example: `can't access tty; job control turned off`)
- no normal login prompt
- `journalctl -xb` returning `No entries`
- `reboot` failing with:

```text
System has not been booted with systemd as init system (PID 1). Can't operate.
```

Those clues matter a lot:

- `No entries` from `journalctl` means `systemd`/`journald` never started
- `reboot` failure means PID 1 is not `systemd`
- this strongly suggests the kernel command line still contains `init=/bin/sh` or similar

## Root cause breakdown

### Root cause A: missing disk still listed in `/etc/fstab`

I had removed previously auto-mounted drives. Boot then waited for those mounts and dropped into failure flow.

### Root cause B: recovery mode parameter left in `cmdline.txt`

At some point, recovery boot was enabled with `init=/bin/sh`. If you forget to remove it, the Pi keeps booting into a bare shell as PID 1 instead of launching `systemd`.

When these two issues combine, recovery becomes confusing: no logs, no services, and commands behave differently.

## Fast diagnosis commands

From the broken shell:

```bash
cat /proc/cmdline
ps aux
mount | grep boot
```

What to look for:

- `cat /proc/cmdline` includes `init=/bin/sh`, `init=/bin/bash`, `systemd.unit=rescue.target`, or `emergency`
- `ps aux` shows only a tiny process list
- boot partition may not be mounted at all

## Recovery steps

## 1) Remove accidental recovery boot parameters

If `/boot/firmware` is empty, that can be normal in this state (no `systemd`, no automatic fstab mounts yet). Mount the boot partition manually:

```bash
mount /dev/mmcblk0p1 /boot/firmware
ls -al /boot/firmware
```

Then edit:

```bash
vi /boot/firmware/cmdline.txt
```

Remove `init=...` and any temporary rescue/emergency options you added.

Important:

- keep `cmdline.txt` as **one single line**
- do not insert extra line breaks

## 2) Handle dirty boot partition warning (if shown)

If mount warns that the volume was not properly unmounted, clean it:

```bash
umount /boot/firmware
fsck.fat -a /dev/mmcblk0p1
mount /dev/mmcblk0p1 /boot/firmware
```

## 3) Fix `/etc/fstab` so missing drives do not block boot

After regaining normal boot (or with root mounted read-write), fix drive entries:

```fstab
UUID=xxxx  /mnt/hdd  ext4  defaults,nofail,x-systemd.device-timeout=5,x-systemd.automount  0  2
```

Meaning:

- `nofail`: continue boot even if drive is absent
- `x-systemd.device-timeout=5`: avoid long boot hangs
- `x-systemd.automount`: mount on first access (reduces timing/race issues)

Then verify before reboot:

```bash
sudo mount -a
```

## 4) Reboot correctly when PID 1 is not systemd

In `init=/bin/sh` mode, plain `reboot` may fail. Use:

```bash
sync
reboot -f
```

## Prevention checklist

- use `UUID=` or `LABEL=` in `/etc/fstab` (avoid `/dev/sdX` names)
- add `nofail` to non-critical external drives
- run `mount -a` after every `fstab` edit
- always remove temporary `cmdline.txt` recovery parameters after the system is healthy
- always shut down cleanly (`shutdown -h now` / `systemctl poweroff`)

## Minimal emergency playbook (copy/paste)

```bash
# Inspect current boot mode
cat /proc/cmdline

# Mount boot partition manually if needed
mount /dev/mmcblk0p1 /boot/firmware

# Edit and remove init=/bin/sh from cmdline.txt
vi /boot/firmware/cmdline.txt

# Optional: repair dirty FAT flag
umount /boot/firmware
fsck.fat -a /dev/mmcblk0p1

# Reboot when systemd is not PID 1
sync
reboot -f
```

After normal boot returns, fix `/etc/fstab` with `nofail` and validate with `mount -a`.

