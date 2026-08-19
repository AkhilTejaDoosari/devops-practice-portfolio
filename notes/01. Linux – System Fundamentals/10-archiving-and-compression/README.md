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

# Archiving and Compression

> **Layer:** L5 — Tools & Files   
> **Depends on:** [03 Working with Files](../03-working-with-files/README.md) — you need cp and mv before you need tar   
> **Used in production when:** Backing up the webstore before a deploy, compressing old logs to free disk space, transferring the entire project to a new server in one file   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. Archiving vs compression](#1-archiving-vs-compression)
- [2. tar — the standard tool](#2-tar--the-standard-tool)
- [3. gzip — compressing single files](#3-gzip--compressing-single-files)
- [4. Reading compressed files without extracting](#4-reading-compressed-files-without-extracting)
- [5. zip and unzip](#5-zip-and-unzip)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

Before every deploy, you archive the current state of the webstore. Before rotating logs, you compress last month's access log. When you need to move the entire project to a new server, you pack it into one file and transfer it. These are not optional practices — they are the habits that let you recover when something goes wrong. This file covers two distinct operations that are often confused: **archiving** (combining multiple files into one, no size reduction) and **compression** (reducing a file's size). `tar` archives. `gzip` compresses. Used together — `tar.gz` — you get both.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       tar gzip zcat zip — pack, compress, and restore
  L4  Config
  L3  State & Debug  ← /var/log — the logs you compress live here
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

Archiving is the safety net under every other layer. Before you edit a config at L4 or restart a service at L1, you tar the project. If something breaks, you restore from the archive.

---

## 1. Archiving vs compression

| Tool | What it does | Output |
|---|---|---|
| `tar` | Combines files into one archive — no compression | `.tar` |
| `gzip` | Compresses a single file — replaces original | `.gz` |
| `tar + gzip` | Archives and compresses in one step — the Linux standard | `.tar.gz` |
| `zip` | Archives and compresses — cross-platform | `.zip` |

**The rule on Linux servers:** use `tar.gz`. It preserves file permissions, ownership, symlinks, and directory structure — everything you need to restore a backup to an identical state. `zip` does not preserve Unix permissions reliably, which matters when your webstore has carefully set `chmod` values.

---

## 2. tar — the standard tool

`tar` reads like a sentence — what to do, what to name the result, what to include.

**The flags you use constantly:**

| Flag | Full form | Meaning |
|---|---|---|
| `c` | --create | Create a new archive |
| `x` | --extract | Extract from an archive |
| `t` | --list | List contents without extracting |
| `z` | --gzip | Compress or decompress with gzip |
| `v` | --verbose | Print each file as it is processed |
| `f` | --file | Next argument is the archive filename — always required |
| `-C` | --directory | Extract to a specific directory |

Flag order: `-czvf archive.tar.gz source/` — flags first, archive name second, source last.

**Create a compressed archive:**

```bash
# Archive + compress the webstore
tar -czvf webstore-backup.tar.gz ~/webstore/
# webstore/
# webstore/config/
# webstore/config/webstore.conf
# webstore/logs/access.log
# ...

# With a timestamp — essential when keeping multiple backups
tar -czvf webstore-backup-$(date +%Y-%m-%d).tar.gz ~/webstore/
# Creates: webstore-backup-2025-04-05.tar.gz
```

**List contents without extracting — always do this before extracting:**

```bash
tar -tzvf webstore-backup-2025-04-05.tar.gz
# drwxr-xr-x akhil/webstore-team    0  2025-04-05  webstore/
# -rw-r----- akhil/webstore-team  128  2025-04-05  webstore/config/webstore.conf
# -rw-rw---- akhil/webstore-team 2.4K  2025-04-05  webstore/logs/access.log
```

Note permissions and ownership are preserved. This confirms the archive contains what you expect before extracting.

**Extract:**

```bash
# Extract to current directory
tar -xzvf webstore-backup-2025-04-05.tar.gz

# Extract to a specific directory — always safer
tar -xzvf webstore-backup-2025-04-05.tar.gz -C /tmp/restore/

# Extract a single file from the archive
tar -xzvf webstore-backup-2025-04-05.tar.gz webstore/config/webstore.conf
```

`-C` is important — without it tar extracts relative to your current directory. With it you control exactly where things land.

---

## 3. gzip — compressing single files

`gzip` compresses one file and replaces it with a `.gz` version. The original file is gone after compression.

```bash
# Compress last month's log — original replaced
gzip ~/webstore/logs/access.log.old
ls -lh ~/webstore/logs/
# -rw-rw-r-- akhil webstore-team 312K access.log.old.gz
# (was 1.8M — 80% reduction typical for log files)

# Keep the original — do not replace it (-k = --keep)
gzip -k ~/webstore/logs/access.log

# Maximum compression (-9 = slowest but smallest)
gzip -9 ~/webstore/logs/error.log.old

# Decompress — restores original file
gunzip ~/webstore/logs/access.log.old.gz
```

Log files compress extremely well — 60-85% reduction — because they contain repetitive text.

---

## 4. Reading compressed files without extracting

You do not have to decompress a log to search it. These commands work directly on `.gz` files:

```bash
# Print entire compressed log to terminal
zcat ~/webstore/logs/access.log.gz

# Page through it
zless ~/webstore/logs/access.log.gz

# Search inside without extracting
zcat ~/webstore/logs/access.log.gz | grep '500'

# Count errors in compressed log
zcat ~/webstore/logs/access.log.gz | grep -c '500'
# 2
```

This is the pattern for historical log analysis. Keep old logs compressed to save disk space, use `zcat` to query them without decompressing to disk.

---

## 5. zip and unzip

Use `zip` when sharing files with Windows systems or APIs that expect `.zip`. Between Linux servers, use `tar.gz`.

```bash
# Zip specific files
zip webstore-logs.zip ~/webstore/logs/access.log ~/webstore/logs/error.log

# Zip a directory recursively (-r = --recurse-paths)
zip -r webstore-config.zip ~/webstore/config/

# List contents without extracting
unzip -l webstore-config.zip

# Extract
unzip webstore-config.zip

# Extract to specific directory (-d = destination)
unzip webstore-config.zip -d /tmp/restore/
```

---

## On the webstore

This is the backup workflow you run before every significant change.

```bash
# Step 1 — create a timestamped backup before the deploy
tar -czvf ~/webstore/backup/webstore-$(date +%Y-%m-%d-%H%M).tar.gz \
    --exclude='~/webstore/backup' \
    ~/webstore/

# Step 2 — verify the archive — list contents and check for key files
tar -tzvf ~/webstore/backup/webstore-2025-04-05-0914.tar.gz | grep 'webstore.conf'
# -rw-r----- akhil/webstore-team 128 2025-04-05 webstore/config/webstore.conf

# Step 3 — check the backup size is reasonable
ls -lh ~/webstore/backup/
# -rw-r--r-- akhil akhil 42K webstore-2025-04-05-0914.tar.gz

# Step 4 — make the change (e.g. edit nginx config)
# ...

# Step 5 — if something breaks, restore
tar -xzvf ~/webstore/backup/webstore-2025-04-05-0914.tar.gz -C /tmp/restore/
ls /tmp/restore/webstore/
# config/  logs/  frontend/  api/  db/  backup/

# Step 6 — compress old logs monthly to free disk space
find ~/webstore/logs/ -name "*.log" -mtime +30 -exec gzip {} \;
ls -lh ~/webstore/logs/
# access.log.gz  error.log.gz  access.log (current month — not compressed)
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `tar: Removing leading / from member names` | tar strips absolute paths by default | Expected behaviour — extract to a specific dir with `-C /target/` |
| Extracted files have wrong permissions | Used `zip` instead of `tar.gz` | Use `tar.gz` for Linux backups — zip does not preserve Unix permissions |
| Archive is missing files | `--exclude` pattern too broad | Test with `-t` list before extracting to see what is inside |
| `gzip: access.log: already has .gz suffix` | Trying to compress an already compressed file | Check with `file access.log.gz` — already compressed |
| `tar -xzvf` extracts to current directory and overwrites files | Missing `-C` flag | Always use `-C /tmp/restore/` to extract to a safe location first |
| Disk fills up with old `.tar.gz` files | No backup rotation | Add `find backup/ -mtime +30 -name "*.tar.gz" -delete` after creating new backup |

---

## Daily commands

| Command | What it does |
|---|---|
| `tar -czvf <archive>.tar.gz <source>` | Create compressed archive |
| `tar -tzvf <archive>.tar.gz` | List archive contents without extracting |
| `tar -xzvf <archive>.tar.gz -C <dir>` | Extract to a specific directory |
| `tar -czvf backup-$(date +%Y-%m-%d).tar.gz <source>` | Create timestamped backup |
| `gzip <file>` | Compress a file — original replaced by .gz |
| `gzip -k <file>` | Compress keeping the original |
| `gunzip <file>.gz` | Decompress a .gz file |
| `zcat <file>.gz \| grep <pattern>` | Search inside compressed file without extracting |
| `zless <file>.gz` | Page through compressed file |
| `zip -r <archive>.zip <dir>` | Zip a directory for cross-platform sharing |

---

→ **Interview questions for this topic:** [99-interview-prep → Archiving](../99-interview-prep/README.md#archiving)
