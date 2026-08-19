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

# Working with Files

> **Layer:** L5 — Tools & Files   
> **Depends on:** [02 Basics](../02-basics/README.md) — you need `ls`, `cd`, and `pwd` before working with files   
> **Used in production when:** Backing up a config before changing it, writing content into a file from the terminal, reading a log without opening an editor, creating the webstore directory structure   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. Create and Inspect Files](#1-create-and-inspect-files)
- [2. Writing Content into Files](#2-writing-content-into-files)
- [3. Copying Files — cp](#3-copying-files--cp)
- [4. Moving and Renaming Files — mv](#4-moving-and-renaming-files--m)v
- [5. Deleting Files — rm](#5-deleting-files--rm)
- [6. Viewing File Contents](#6-viewing-file-contents)
- [7. Previewing File Sections — head and tail](#7-previewing-file-sections--head-and-tail)
- [8. Symbolic Links — ln](#8-symbolic-links--ln)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

On a Linux server, everything is a file. Config files, log files, scripts, sockets, devices — all of them live in the filesystem and all of them are operated on with the same small set of commands. This file covers creating files, reading their contents, copying and moving them, deleting them, and understanding symlinks. These are not beginner exercises — they are the operations you perform every single time you work on a server. Getting comfortable with `-i` and `-v` flags here will prevent data loss in production.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       cp · mv · rm · cat · less · head · tail · touch · ln
  L4  Config
  L3  State & Debug
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

The webstore directory from file 02 is now a skeleton of empty folders. This file fills it — writing config files into it, reading its logs, backing it up before changes. Every tool from file 04 onward operates on files you know how to handle after this.

---

## 1. Create and Inspect Files

`touch` creates an empty file if it does not exist, or updates the last-modified timestamp if it does. On a server you use it to create placeholder files or initialise log files before a service starts writing to them.

`file` examines the actual content of a file and reports what type it is — not based on the extension, but based on the bytes inside. Linux does not care about extensions. A file called `server.conf` could contain anything — `file` tells you what it actually is.

`stat` (status) shows the full metadata of a file — exact size, all three timestamps, permissions in both numeric and symbolic form, and the inode number. When a deployment goes wrong and you need to know exactly when a config file was last changed, `stat` gives you the answer down to the second.

| Command | Full form | What it does | When you reach for it |
|---|---|---|---|
| `touch <file>` | — | Create an empty file or update its timestamp | Creating `~/webstore/logs/access.log` before nginx starts writing to it |
| `file <file>` | — | Report the actual content type of a file | Confirming a downloaded binary is an ELF executable, not a corrupted file |
| `stat <file>` | Status | Show full metadata — size, all timestamps, permissions, inode | Finding the exact second `webstore.conf` was last modified during an incident |

**What `stat` output tells you:**

```bash
stat ~/webstore/config/webstore.conf
# File: webstore.conf
# Size: 128        Blocks: 8    IO Block: 4096   regular file
# Inode: 524291    Links: 1
# Access: (0644/-rw-r--r--)  Uid: (1000/akhil)  Gid: (33/www-data)
# Access: 2025-04-05 09:12:01   ← last read
# Modify: 2025-04-05 08:47:33   ← last content change
# Change: 2025-04-05 08:47:33   ← last metadata change (permissions etc)
```

Three timestamps — **Access** (last read), **Modify** (last content change), **Change** (last metadata change including permissions). If `Modify` and `Change` differ, someone changed permissions without touching the content. That is worth knowing during an incident.

---

## 2. Writing Content into Files

Two operators write content into files from the command line — `>` and `>>`.     
Knowing the difference prevents you from accidentally wiping a file you meant to append to.

| Operator | What it does | When you reach for it |
|---|---|---|
| `echo "text" > <file>` | Write text to a file — **overwrites** entirely if file exists | Creating `webstore.conf` from scratch with a single line |
| `echo "text" >> <file>` | Append text to a file — adds to the end, never overwrites | Adding a new config entry to an existing file without disturbing the rest |

```bash
# Create webstore.conf with initial content
echo "db_host=webstore-db" > ~/webstore/config/webstore.conf
echo "db_port=5432" >> ~/webstore/config/webstore.conf
echo "api_port=8080" >> ~/webstore/config/webstore.conf

# Verify what was written
cat ~/webstore/config/webstore.conf
# db_host=webstore-db
# db_port=5432
# api_port=8080
```

 Use `>>` **(append)** when you mean to add.  
Use `>` **(overwrite)** only when you mean to replace everything.   
 **`>` overwrites without warning.** `echo "new" > webstore.conf` replaces the entire file with one word.  

> **Rule of Thumb:**    
Use `echo` for quickly injecting small pieces of data or appending single lines to files via scripts.   
For creating or editing large, complex configuration files, use a text editor like `vim`.
---

## 3. Copying Files — `cp`

`cp` (copy) copies a file or directory. The original stays in place.

| Command | Full form | What it does |
|---|---|---|
| `cp <src> <dest>` | Copy | Copy a file to a new location or name |
| `cp -r <src> <dest>` | Copy --recursive | Copy a directory and everything inside it |
| `cp -i <src> <dest>` | Copy --interactive | Ask before overwriting — prompt if destination exists |
| `cp -v <src> <dest>` | Copy --verbose | Print each file as it is copied |
| `cp -riv <src> <dest>` | Copy --recursive --interactive --verbose | Gold standard for directory copies |

**`-i` prevents silent overwrites:**

```bash
# Without -i — silently overwrites webstore.conf if it already exists
cp webstore.conf /etc/webstore/webstore.conf

# With -i — pauses and asks first
cp -i webstore.conf /etc/webstore/webstore.conf
# cp: overwrite '/etc/webstore/webstore.conf'? y
```

**`-v` confirms the copy actually happened:**

```bash
# Without -v — no output, no confirmation
cp -r ~/webstore ~/webstore-backup

# With -v — prints every file as it copies
cp -rv ~/webstore ~/webstore-backup
# ~/webstore -> ~/webstore-backup
# ~/webstore/config/webstore.conf -> ~/webstore-backup/config/webstore.conf
# ~/webstore/logs/access.log -> ~/webstore-backup/logs/access.log
```

**Gold standard — use this before every production change:**

```bash
cp -riv ~/webstore ~/webstore-backup
```

`-r` handles directories, `-i` won't silently overwrite, `-v` shows every file as it copies. You get safety and visibility in one command.

---

## 4. Moving and Renaming Files — `mv`

`mv` (move) handles both moving and renaming — they are the same operation. If the destination is a different path, the file moves. If the destination is a new name in the same directory, the file is renamed. The original is gone either way.

| Command | Full form | What it does |
|---|---|---|
| `mv <src> <dest>` | Move | Move or rename a file or directory |
| `mv -i <src> <dest>` | Move --interactive | Ask before overwriting the destination |
| `mv -v <src> <dest>` | Move --verbose | Print what moved and where it landed |

```bash
# Move — relocate to a different directory
mv webstore.conf /etc/webstore/webstore.conf

# Rename — new name in the same directory
mv webstore.conf webstore.conf.bak

# With -v confirmation
mv -v webstore.conf.bak webstore.conf.backup
# 'webstore.conf.bak' -> 'webstore.conf.backup'
```

`mv` has no `-r` flag because it already handles directories natively.

> **`mv` vs `cp` + `rm`** — always use `mv` to relocate files. `mv` preserves all metadata including timestamps and ownership. `cp` + `rm` creates a new file and loses the original timestamps.

**Gold standard combinations:**

| Situation | Command |
|---|---|
| Back up a directory before changing it | `cp -riv ~/webstore ~/webstore-backup` |
| Copy a single config file safely | `cp -iv webstore.conf /etc/webstore/webstore.conf` |
| Rename a file with confirmation | `mv -iv webstore.conf webstore.conf.bak` |

---

## 5. Deleting Files — `rm`

`rm` (remove) deletes files permanently. There is no trash, no recycle bin, no undo. The habit to build now: always run `ls <path>` first to confirm exactly what you are about to delete.

| Command | Full form | What it does | When you reach for it |
|---|---|---|---|
| `rm <file>` | Remove | Delete a file permanently | Removing a stale lock file blocking a service restart |
| `rm -i <file>` | Remove --interactive | Prompt before each deletion | Deleting multiple files when you want to confirm each one |
| `rm -r <dir>` | Remove --recursive | Delete a directory and all its contents | Removing a build output directory before a fresh deploy |
| `rm -f <file>` | Remove --force | No prompt, no error if file does not exist | Deleting temp files in scripts where the file may or may not be there |
| `rm -rf <dir>` | Remove --recursive --force | Delete an entire directory tree with no confirmation | Wiping a temp directory — verify the path first, every time |

> **`rm -rf` has no confirmation and no undo.** `rm -rf /webstore` and `rm -rf ~/webstore` are completely different operations. Always run `ls <path>` first to confirm what is there before deleting it.

---

## 6. Viewing File Contents

Reading file contents from the terminal is something you do constantly on a server — checking config values, reading logs, verifying that a write operation worked.

| Command | Full form | What it does | When you reach for it |
|---|---|---|---|
| `cat <file>` | Concatenate | Print entire file to terminal | Reading `webstore.conf` to check the current db_host value |
| `cat -n <file>` | Concatenate --number | Print with line numbers | When an error references a specific line number in a config |
| `less <file>` | — | Scroll through a file page by page | Any file too long to read in one screen — logs, long configs |
| `tac <file>` | — (cat backwards) | Print file in reverse line order | Reading a log from bottom to top when newest entries matter most |

Inside `less`: `Space` scroll down, `b` scroll back, `/pattern` search, `n` next match, `q` quit. For a 2 GB log file, `less` handles it instantly — `cat` would flood your terminal.

---

## 7. Previewing File Sections — `head` and `tail`

When debugging a live service you rarely need the whole file. You need the last 50 lines where the error happened, or the first 10 lines of a config to confirm the format.

| Command | Full form | What it does | When you reach for it |
|---|---|---|---|
| `head <file>` | — | Show first 10 lines | Checking the header of a log file to confirm its format |
| `head -n <N> <file>` | head --lines | Show first N lines | `head -n 3 webstore.conf` — reading just the first three config entries |
| `tail <file>` | — | Show last 10 lines | Checking the most recent entries in `access.log` after a request |
| `tail -n <N> <file>` | tail --lines | Show last N lines | `tail -n 50 error.log` — reading the last 50 lines during an incident |
| `tail -f <file>` | tail --follow | Follow live — print new lines as they are written | Watching `access.log` in real time while testing a webstore endpoint |

`tail -f` is the command for watching a live service. Open a second terminal, run `tail -f ~/webstore/logs/access.log`, then make a request — you see the log entry appear the moment it is written.

---

## 8. Symbolic Links — `ln`

A symbolic link (symlink) is a pointer to another file or directory. The file exists in one place — the symlink makes it appear somewhere else. Deleting the symlink does not delete the file. Deleting the file leaves a broken symlink.

nginx uses symlinks to manage site configs — the actual config lives in `sites-available/`, and a symlink in `sites-enabled/` points to it. Enabling a site means creating the symlink. Disabling means removing it.

```bash
# Create a symlink
ln -s (s = symbolic) /etc/nginx/sites-available/webstore /etc/nginx/sites-enabled/webstore

# What it looks like in ls -l output
ls -la /etc/nginx/sites-enabled/
# lrwxrwxrwx 1 root root 34 Apr 5 09:00 webstore -> ../sites-available/webstore
# ^                                               ^
# l = symlink                                     -> points to the real file
```

| Command | Full form | What it does |
|---|---|---|
| `ln -s <target> <link>` | Link --symbolic | Create a symlink pointing to target |
| `ls -la` | List --long --all | See symlinks — shown with `l` prefix and `->` arrow |
| `readlink <link>` | — | Show where a symlink points |
| `unlink <link>` | — | Remove a symlink without deleting the target |

---

## On the webstore

The directory structure exists from file 02. Now you write content into it, back it up, and verify every file is correct before moving forward.

```bash
# Step 1 — write the webstore config file
echo "frontend_port=80" >> ~/webstore/config/webstore.conf
echo "api_host=webstore-api" >> ~/webstore/config/webstore.conf
echo "api_port=8080" >> ~/webstore/config/webstore.conf
echo "db_host=webstore-db" >> ~/webstore/config/webstore.conf
echo "db_port=5432" >> ~/webstore/config/webstore.conf
echo "db_name=webstore" >> ~/webstore/config/webstore.conf
echo "env=production" >> ~/webstore/config/webstore.conf

# Step 2 — verify it
cat ~/webstore/config/webstore.conf
frontend_port=80
api_host=webstore-api
api_port=8080
db_host=webstore-db
db_port=5432
db_name=webstore
env=production

# Step 3 — create a placeholder log file before nginx starts
touch ~/webstore/logs/access.log
touch ~/webstore/logs/error.log

# Step 4 — check the exact metadata on the config file
stat ~/webstore/config/webstore.conf
# Modify: 2025-04-05 09:14:22  ← you will use this during incident triage

# Step 5 — back up the entire project before making any changes
cp -riv ~/webstore ~/webstore-backup
# ~/webstore -> ~/webstore-backup
# ~/webstore/config/webstore.conf -> ~/webstore-backup/config/webstore.conf
# ~/webstore/logs/access.log -> ~/webstore-backup/logs/access.log

# Step 6 — verify the backup has the same structure
ls -la ~/webstore-backup/
# config/  logs/  frontend/  api/  db/  backup/

# Step 7 — read only the first 3 lines of the config
head -n 3 ~/webstore/config/webstore.conf
frontend_port=80
api_host=webstore-api
api_port=8080
```

The webstore now has a real config file and placeholder log files. The backup exists. From here, file 04 will search and filter this config. File 05 will edit it in-place. File 07 will let you open and edit it in vim.

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `cp: cannot stat: No such file or directory` | Source path does not exist or is misspelled | `ls` the source directory to confirm the exact filename |
| `cp: omitting directory` | Used `cp` without `-r` on a directory | Add `-r` — `cp -riv <src> <dest>` |
| `mv: cannot move: Permission denied` | You do not own the destination directory | Check with `ls -ld <dest>` — you may need `sudo` |
| `>` wiped a file you meant to append to | Used `>` instead of `>>` | Restore from backup — this is why you run `cp -riv` before every change |
| `rm: cannot remove: Is a directory` | Used `rm` without `-r` on a directory | Use `rm -r <dir>` — add `-f` only if you are certain |
| `tail -f` shows nothing new | The service is not writing to that file | Check `systemctl status <service>` — the service may have stopped |
| Symlink shows `->` but target is missing | The file the symlink points to was deleted | Broken symlink — recreate the target or remove the symlink with `unlink` |

---

## Daily commands

| Command | What it does |
|---|---|
| `touch <file>` | Create an empty file or update its timestamp |
| `stat <file>` | Show full metadata — size, all timestamps, permissions |
| `cat <file>` | Print entire file contents to terminal |
| `less <file>` | Scroll through a large file page by page |
| `tail -f <file>` | Follow a file live — new lines appear as they are written |
| `head -n <N> <file>` | Show the first N lines of a file |
| `cp -riv <src> <dest>` | Copy a directory safely with confirmation and visibility |
| `mv -iv <src> <dest>` | Move or rename with confirmation and visibility |
| `ln -s <target> <link>` | Create a symbolic link |
| `rm -i <file>` | Delete a file with a confirmation prompt |

---

→ **Interview questions for this topic:** [99-interview-prep → Working with Files](../99-interview-prep/README.md#working-with-files)
