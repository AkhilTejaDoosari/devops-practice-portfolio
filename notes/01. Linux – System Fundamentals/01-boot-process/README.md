[Home](../README.md) |
[Boot](../01-boot-process/README.md) |
[Basics](../02-basics/README.md) |
[Files](../03-working-with-files/README.md) |
[Filters](../04-filter-commands/README.md) |
[sed](../05-sed-stream-editor/README.md) |
[awk](../06-awk/README.md) |
[vim](../07-text-editor/README.md) |
[Users](../08-user-and-group-management/README.md) |
[Permissions](../09-file-ownership-and-permissions/README.md) |
[Archive](../10-archiving-and-compression/README.md) |
[Packages](../11-package-management/README.md) |
[Services](../12-service-management/README.md) |
[Networking](../13-networking/README.md) |
[Logs](../14-logs-and-debug/README.md) |
[Interview](../99-interview-prep/README.md)

---

# Boot Process

> **Layer:** L0 — Kernel & Hardware   
> **Depends on:** Nothing this is the first file   
> **Used in production when:** A server won't boot, you need to know which stage failed and where to look

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [The Boot Sequence](#the-boot-sequence)
- [1. Firmware — BIOS and UEFI](#1-firmware--bios-and-uefi)
- [2. GRUB2 — The Bootloader](#2-grub2--the-bootloader)
- [3. initramfs — Why it exists](#3-initramfs--why-it-exists)
- [4. The Kernel](#4-the-kernel)
- [5. systemd — PID 1](#5-systemd--pid-1)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

The boot process is a relay race. Six stages run in a fixed order — each one does its job and hands off to the next. If any stage fails, the race stops exactly there and the error tells you which stage broke. Understanding this sequence means you never blindly stare at a blank screen — you read the error, identify the stage, and know exactly where to look. This file gives you that map.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files
  L4  Config
  L3  State & Debug
  L2  Networking
  L1  Process Manager  ← systemd starts here, at the end of boot
  L0  Kernel & Hardware  ← this file lives here
```

Everything above L0 only exists because the boot process assembled it.
BIOS wakes the hardware. GRUB loads the kernel. The kernel starts systemd.
systemd builds everything above it. If L0 fails, nothing above it can exist.

---

## The Boot Sequence

```
Power ON
   │
   ▼
┌─────────────────────────────────────────────────────────────────┐
│  BIOS / UEFI  (firmware on the motherboard chip)                │
│  job: run POST, find bootable disk, load GRUB into RAM          │
│  breaks as: blank screen · beep codes · "no bootable device"    │
│  fix: physical check — RAM seated, disk connected, BIOS order   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  GRUB2  (Grand Unified Bootloader)                              │
│  job: show boot menu, load kernel + initramfs, hand off         │
│  breaks as: grub rescue> prompt · "no such partition" error     │
│  fix: boot live USB → chroot → grub-install → update-grub       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Kernel  (/boot/vmlinuz-*)                                      │
│  job: load drivers, mount real /, hand off to systemd           │
│  breaks as: kernel panic — text wall on screen                  │
│  fix: dmesg | less — read the last lines before the panic       │
│       boot older kernel from GRUB menu as fallback              │
└──────────────────────────────┬──────────────────────────────────┘
                               │  ← initramfs used here, then discarded
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  systemd  (PID 1)                                               │
│  job: read unit files, start services, reach boot target        │
│  breaks as: emergency shell · failed units listed on screen     │
│  fix: journalctl -xb · systemctl list-units --state=failed      │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
                        Login prompt ✓
```

Each error message tells you exactly which stage broke.
`grub rescue>` → GRUB failed, don't look at the kernel.
`kernel panic` → GRUB worked, look at drivers or the filesystem.
`emergency shell` → kernel worked, look at systemd unit files.

---

## 1. Firmware — BIOS and UEFI

The firmware is the first thing that runs when a machine gets power.
It lives on a chip on the motherboard — not Linux, not an OS.
Its only job is to wake up the hardware and find something bootable.

**What it does:**
- Runs **POST** (Power-On Self Test) — checks CPU, RAM, and storage are present
- Finds a bootable disk based on the boot order you set in BIOS settings
- Loads the bootloader (GRUB) from that disk into RAM
- Steps aside — firmware is done

**BIOS vs UEFI — what you actually need to know:**

| | BIOS | UEFI |
|---|---|---|
| Age | Legacy — 1970s design | Modern standard — what all servers use now |
| Disk table | Works with MBR | Works with GPT |
| Max disk size | 2 TB limit | No practical limit |
| Boot speed | Slower | Faster — loads more directly |

UEFI is on every modern server. You will only see BIOS on hardware older than ~2012.
GPT is the partition table standard. MBR is legacy — 2 TB limit, max 4 partitions.

---

## 2. GRUB2 — The Bootloader

GRUB2 (Grand Unified Bootloader 2) is the first Linux-aware software that runs.
Firmware knows nothing about Linux — it just finds a disk.
GRUB knows exactly where the kernel lives and how to load it.

**What GRUB does:**
- Displays the OS selection menu (dual-boot machines)
- Loads the kernel (`/boot/vmlinuz-*`) into RAM
- Loads **initramfs** (`/boot/initrd.img-*`) alongside the kernel
- Passes control to the kernel — GRUB's job is done in seconds

**The two files rule — this is what you actually touch:**

| File | What it is | What you do |
|---|---|---|
| `/etc/default/grub` | Human-editable settings | **Edit this one** |
| `/boot/grub/grub.cfg` | Auto-generated final config | **Never touch this** |

After editing `/etc/default/grub`, regenerate the final config:
```bash
sudo update-grub
```

**Common things you change in `/etc/default/grub`:**
```bash
GRUB_TIMEOUT=5           # seconds to show the menu before auto-booting
GRUB_DEFAULT=0           # which menu entry boots by default (0 = first)
GRUB_CMDLINE_LINUX=""    # extra parameters passed to the kernel at boot
                         # example: "net.ifnames=0" forces old-style eth0 naming
```

---

## 3. initramfs — Why it exists

The kernel needs drivers to mount the real root filesystem (`/`).
But those drivers live on the real root filesystem.
Classic chicken-and-egg problem.

**initramfs** breaks the deadlock. It is a tiny filesystem loaded into RAM
by GRUB alongside the kernel. It contains just enough drivers to mount
the real disk. Once `/` is mounted, initramfs is discarded and forgotten.

You rarely touch it directly. It rebuilds automatically when you update the kernel.

```bash
# See your initramfs files
ls -lh /boot/initrd.img-*
# -rw-r--r-- 1 root root 52M Apr 5 09:12 /boot/initrd.img-6.5.0-26-generic
```

---

## 4. The Kernel

The kernel is the only software that talks directly to hardware.
Once GRUB hands control to it, the kernel takes over completely.

**What the kernel does at boot:**
1. Decompresses itself into RAM
2. Uses initramfs to access storage drivers
3. Mounts the real root filesystem (`/dev/sda1` or equivalent)
4. Discards initramfs
5. Starts systemd as PID 1 — the first user-space process

The kernel binary lives at `/boot/vmlinuz-*`.
The `vmlinuz` name is historical — "vm" for virtual memory, "linuz" for Linux, "z" for compressed.

---

## 5. systemd — PID 1

systemd is the first process the kernel starts. It always gets **PID 1** —
process ID number one. Every other process on the system is a child of systemd.
If systemd dies, the system goes down.

**What systemd manages:**
- Starting and stopping all services (nginx, sshd, cron, docker)
- Boot targets — the state the system should reach after boot
- Logging via `journald` — all service logs flow through here
- Mounts, sockets, timers

**Unit types — the files systemd reads:**

| Unit | Extension | Purpose |
|---|---|---|
| Service | `.service` | A background daemon — nginx, sshd, postgresql |
| Target | `.target` | A group of units — defines what "booted" means |
| Socket | `.socket` | Starts a service when a connection arrives on a port |
| Mount | `.mount` | A filesystem mount point managed by systemd |
| Timer | `.timer` | A scheduled job — modern replacement for cron |

**Boot targets — the two you need to know:**

| Target | Old runlevel | What it means |
|---|---|---|
| `multi-user.target` | 3 | Full system, networking up, no GUI — this is every server |
| `graphical.target` | 5 | Full system, networking up, GUI running — desktops |
| `rescue.target` | 1 | Minimal, single user, no networking — recovery mode |
| `emergency.target` | — | Bare minimum root shell — last resort |

```bash
# See what target the system boots into
systemctl get-default
# multi-user.target

# Change the default boot target
sudo systemctl set-default multi-user.target
```

---

## On the webstore

At this stage the webstore does not exist yet — there is nothing to deploy.
But the boot process is what runs every time the server starts, and knowing
it means you can answer the first question in any incident:
**is the machine actually up, and did it come up cleanly?**

```bash
# Step 1 — SSH into the server. Confirm it came up and how long it's been running
uptime
# 09:14:22 up 2:03, 1 user, load average: 0.08, 0.03, 0.01

# Step 2 — Check what kernel is running
uname -r
# 6.5.0-26-generic

# Step 3 — Check for any errors during this boot
journalctl -b --priority=err
# (should be empty on a clean boot — any output means something failed)

# Step 4 — Check what systemd brought up
systemctl list-units --type=service --state=running
# shows every service currently running

# Step 5 — Check the boot partition — kernel, initramfs, GRUB files
ls -lh /boot
# vmlinuz-6.5.0-26-generic
# initrd.img-6.5.0-26-generic
# grub/

# Step 6 — View the GRUB settings
cat /etc/default/grub
```

When you install nginx in file 11 and enable it with systemd in file 12,
it will start automatically on every boot because systemd reads the unit
file during boot and starts every enabled service. That process starts here.

---

## What breaks

| Symptom | Stage that broke | First command |
|---|---|---|
| Blank screen, beep codes | BIOS/UEFI | Physical check — RAM, cables, boot order in BIOS |
| `grub rescue>` prompt | GRUB | Boot live USB → `chroot` → `grub-install /dev/sda` → `update-grub` |
| `error: no such partition` | GRUB | Disk UUID changed — boot live USB, update `/boot/grub/grub.cfg` |
| Kernel panic text on screen | Kernel | `journalctl -b -1` — logs from previous boot. Try older kernel from GRUB menu |
| Emergency shell on boot | systemd | `journalctl -xb` — find the failed unit. Fix it then `systemctl reboot` |
| Boots but service missing | systemd | `systemctl list-units --state=failed` — find what didn't start |
| System boots slowly | systemd | `systemd-analyze blame` — shows which service took the longest |

---

## Daily commands

| Command | What it does |
|---|---|
| `uname -r` | Show the kernel version currently running |
| `systemctl get-default` | Show which target the system boots into |
| `systemctl list-units --state=failed` | Show every unit that failed during boot |
| `journalctl -b` | Show all logs from the current boot |
| `journalctl -b -1` | Show all logs from the previous boot — useful after a crash |
| `journalctl -b --priority=err` | Show only errors from current boot |
| `dmesg \| less` | Show kernel hardware messages — look here after a crash |
| `ls -lh /boot` | See kernel, initramfs, and GRUB files |
| `sudo update-grub` | Regenerate `/boot/grub/grub.cfg` after editing `/etc/default/grub` |

---

→ **Interview questions for this topic:** [99-interview-prep → Boot Process](../99-interview-prep/README.md#boot-process)
