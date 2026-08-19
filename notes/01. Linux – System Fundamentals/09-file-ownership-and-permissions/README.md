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

# File Ownership & Permissions

> **Layer:** L4 — Config   
> **Depends on:** [08 Users & Groups](../08-user-and-group-management/README.md) — you need users and groups before you can assign ownership   
> **Used in production when:** nginx cannot read the webstore config, a deploy script cannot write to the logs directory, or you need to lock down files containing database passwords   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. The permission model](#1-the-permission-model)
- [2. Reading ls -l output](#2-reading-ls--l-output)
- [3. Numeric permissions — the octal system](#3-numeric-permissions--the-octal-system)
- [4. chmod — changing permissions](#4-chmod--changing-permissions)
- [5. chown and chgrp — changing ownership](#5-chown-and-chgrp--changing-ownership)
- [6. Special permissions — SUID SGID sticky](#6-special-permissions--suid-sgid-sticky)
- [7. umask — default permissions](#7-umask--default-permissions)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

Every file on a Linux system has an owner, a group, and a set of permissions. These three things together answer one question: who is allowed to do what with this file. This is not abstract security theory. When nginx cannot read the webstore config file, it is a permissions problem. When a deploy script cannot write to the logs directory, it is a permissions problem. When a developer accidentally deletes a shared file, a missing sticky bit is the reason. Understanding permissions is understanding why services fail and how to fix them.

---

## How it fits the stack

```
  L6  You  ← /home/akhil owned by akhil:akhil chmod 700
  L5  Tools & Files
  L4  Config  ← this file lives here
       chmod chown chgrp — control access to every file on the system
  L3  State & Debug
  L2  Networking
  L1  Process Manager  ← nginx runs as www-data — must have read on config
  L0  Kernel & Hardware
```

Permissions connect users (file 08) to the files they can access. Without correct permissions, services fail to read their configs, processes cannot write their logs, and security is broken.

---

## 1. The permission model

Every file has three sets of permissions — owner, group, and others (everyone else).

```
-rw-r--r--  1  akhil  webstore-team  1.2K  webstore.conf
│└────────┘     │      │
│               │      └── group
│               └── owner
└── file type: - = file, d = directory, l = symlink
```

Each set has three bits:

```
OWNER   GROUP   OTHERS
r w x   r w x   r w x
│ │ │
│ │ └── execute: run file as program / enter directory
│ └──── write:   modify file / create+delete files in directory
└────── read:    read file contents / list directory contents
```

**The directory execute bit is the one people miss.** A directory with `r` but no `x` lets you see filenames with `ls` but you cannot `cd` into it or access any files inside. You need `x` to enter a directory.

---

## 2. Reading ls -l output

```bash
ls -lh ~/webstore/
# drwxr-xr-x  2  akhil  webstore-team  4.0K  config/
# drwxrwsr-x  2  akhil  webstore-team  4.0K  logs/
# -rw-r-----  1  akhil  webstore-team   128  config/webstore.conf
```

Decoding `drwxr-xr-x`:
- `d` — directory
- `rwx` — owner (akhil): read, write, enter
- `r-x` — group (webstore-team): read and enter, no write
- `r-x` — others: read and enter, no write

Decoding `-rw-r-----`:
- `-` — regular file
- `rw-` — owner: read and write
- `r--` — group: read only
- `---` — others: no access at all

---

## 3. Numeric permissions — the octal system

Each bit has a value. Add them for each set.

| Value | Bit | Meaning |
|---|---|---|
| 4 | `r` | read |
| 2 | `w` | write |
| 1 | `x` | execute |

**The permissions you use most in production:**

| Octal | Symbolic | Use case |
|---|---|---|
| `600` | `rw-------` | SSH private keys, credential files — owner only |
| `640` | `rw-r-----` | Config files with secrets — owner writes, group reads |
| `644` | `rw-r--r--` | Public config files — owner writes, everyone reads |
| `664` | `rw-rw-r--` | Shared files — owner and group write |
| `700` | `rwx------` | Private scripts and directories |
| `750` | `rwxr-x---` | Directories — owner full, group enters, others nothing |
| `755` | `rwxr-xr-x` | Public directories and executables |
| `770` | `rwxrwx---` | Shared directories — owner and group full access |
| `777` | `rwxrwxrwx` | Full access for everyone — almost never correct in production |

**Reading `640` as three digits:**
`6` = owner gets rw- (4+2=6) · `4` = group gets r-- (4) · `0` = others get --- (0)

---

## 4. chmod — changing permissions

**Octal mode — set exact permissions:**

```bash
# Config file with secrets — owner reads+writes, group reads, others nothing
chmod 640 ~/webstore/config/webstore.conf

# Logs directory — owner and group can write (nginx writes here)
chmod 770 ~/webstore/logs/

# Frontend files — nginx reads these, so others need read
chmod 644 ~/webstore/frontend/index.html
chmod 755 ~/webstore/frontend/

# Deploy script — only owner executes
chmod 700 ~/webstore/api/deploy.sh

# Recursively set entire directory tree
chmod -R 755 ~/webstore/
```

**Symbolic mode — add or remove specific bits:**

```bash
# u = user/owner, g = group, o = others, a = all three
# + adds, - removes, = sets exactly

chmod u+x deploy.sh        # add execute for owner only
chmod o-w webstore.conf    # remove write from others
chmod g+w logs/            # add write for group
chmod o= webstore.conf     # set others to no permissions
chmod a+r index.html       # add read for everyone
```

**Octal vs symbolic — when to use which:**
Octal sets the complete final state — use it when you know exactly what permissions should be.
Symbolic makes a targeted change without resetting everything else — use it when you want to add or remove one bit.

---

## 5. chown and chgrp — changing ownership

```bash
# Change both owner and group
sudo chown akhil:webstore-team ~/webstore/config/webstore.conf

# Change owner only
sudo chown akhil ~/webstore/logs/access.log

# Change group only
sudo chgrp webstore-team ~/webstore/config/

# Change recursively — entire directory tree
sudo chown -R akhil:webstore-team ~/webstore/
```

`sudo` is required — only root can change a file's owner. A regular user can change the group of files they own, but only to groups they belong to.

---

## 6. Special permissions — SUID SGID sticky

**SGID on a directory (the one you use most):**

Any new files created inside inherit the directory's group instead of the creator's primary group. Essential for shared team directories — every file created in `~/webstore/logs/` automatically belongs to `webstore-team` regardless of who created it.

```bash
# Set SGID on logs directory
sudo chmod g+s ~/webstore/logs/
ls -ld ~/webstore/logs/
# drwxrwsr-x  akhil  webstore-team  logs/
#       ^
#       s in group execute position = SGID set
```

**Sticky bit on a directory:**

Only the file's owner, the directory owner, or root can delete files inside — even if the directory is world-writable. `/tmp` always has this set.

```bash
sudo chmod +t ~/webstore/logs/
ls -ld ~/webstore/logs/
# drwxrwxrwt  logs/
#          ^
#          t = sticky bit
```

**SUID on an executable (informational — you read this, rarely set it):**

The file runs with the file owner's privileges regardless of who launches it. Classic example: `/usr/bin/passwd` runs as root so any user can update their own password in `/etc/shadow`.

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x  root  root  /usr/bin/passwd
#    ^
#    s in owner execute position = SUID set
```

---

## 7. umask — default permissions

When a new file or directory is created, Linux starts from a maximum and subtracts the umask.

- Files start at `666` (no execute) → umask `022` → result `644`
- Directories start at `777` → umask `022` → result `755`
- umask `027` → files get `640`, directories get `750`

```bash
# Check current umask
umask
# 0022

# Set more restrictive for current session
umask 027

# Make permanent
echo 'umask 027' >> ~/.bashrc
```

`027` matters for deploy scripts that create config files — default `022` makes them world-readable. Any user on the server could read your database password. `027` means only owner and group can read new files.

---

## On the webstore

This is the complete permission setup for the webstore. Every number has a reason.

```bash
# Step 1 — set ownership — akhil owns, webstore-team is the group
sudo chown -R akhil:webstore-team ~/webstore/

# Step 2 — directories — owner full, group enters and reads, others nothing
chmod 750 ~/webstore/
chmod 750 ~/webstore/config/
chmod 750 ~/webstore/api/
chmod 750 ~/webstore/db/

# Step 3 — logs directory — owner and group write (nginx writes as www-data)
chmod 770 ~/webstore/logs/

# Step 4 — config file with secrets — owner rw, group r, others nothing
chmod 640 ~/webstore/config/webstore.conf

# Step 5 — frontend — nginx needs read, so others need read too
chmod 755 ~/webstore/frontend/
chmod 644 ~/webstore/frontend/index.html

# Step 6 — deploy script — only owner executes
chmod 700 ~/webstore/api/deploy.sh

# Step 7 — SGID on logs — nginx files inherit webstore-team group automatically
sudo chmod g+s ~/webstore/logs/

# Step 8 — verify everything
ls -lah ~/webstore/
# drwxr-x--- akhil webstore-team  .          ← 750
# drwxrwx--- akhil webstore-team  logs/      ← 770
# drwxr-x--- akhil webstore-team  config/    ← 750
# -rw-r----- akhil webstore-team  config/webstore.conf  ← 640
# -rwx------ akhil webstore-team  api/deploy.sh         ← 700
```

**Why `640` on webstore.conf and not `644`:**
`644` would let any user on the server read the config — including the database password. `640` means only `akhil` and members of `webstore-team` can read it. nginx runs as `www-data` which is a member of `webstore-team`, so it can read the config. Random users cannot.

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `nginx: [warn] could not open error log file: Permission denied` | nginx cannot write to the logs directory | `chmod 770 ~/webstore/logs/` and verify www-data is in webstore-team |
| `curl localhost` returns 403 Forbidden | nginx can reach the directory but cannot read the files | `chmod 644` on the HTML files and `chmod 755` on the directory |
| `Permission denied` when running a script | Script lacks execute permission | `chmod u+x script.sh` |
| `chown: changing ownership: Operation not permitted` | Only root can change file owners | `sudo chown user:group file` |
| New files in shared dir owned by wrong group | SGID not set on the directory | `sudo chmod g+s <directory>` |
| `chmod -R 777` used and now it feels wrong | World-writable files are a security risk | Set properly: `chmod -R 750` for dirs, `chmod -R 640` for files |

---

## Daily commands

| Command | What it does |
|---|---|
| `ls -lah <dir>` | See permissions, owner, group for all files including hidden |
| `chmod 640 <file>` | Set exact permissions — owner rw, group r, others nothing |
| `chmod u+x <file>` | Add execute for owner only |
| `chmod -R 755 <dir>` | Set permissions recursively on directory tree |
| `sudo chown <user>:<group> <file>` | Change owner and group |
| `sudo chown -R <user>:<group> <dir>` | Change ownership recursively |
| `sudo chmod g+s <dir>` | SGID — new files inside inherit directory's group |
| `chmod +t <dir>` | Sticky bit — only owners can delete inside |
| `umask` | Show current default permissions |
| `ls -li <dir>` | Show inode numbers alongside file details |

---

→ **Interview questions for this topic:** [99-interview-prep → Permissions](../99-interview-prep/README.md#permissions)
