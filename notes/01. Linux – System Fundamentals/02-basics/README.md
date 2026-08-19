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

# Linux Basics

> **Layer:** L6 — You   
> **Depends on:** [01 Boot Process](../01-boot-process/README.md) — you should know what booted before you navigate it   
> **Used in production when:** You SSH into any server — familiar or not — and need to know where you are, what is running, and what the machine looks like before touching anything

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. Directory Navigation](#1-directory-navigation)
- [2. Listing Directory Contents](#2-listing-directory-contents)
- [3. Terminal Essentials](#3-terminal-essentials)
- [4. System Information](#4-system-information)
- [5. Getting Help](#5-getting-help)
- [6. Kernel and System Info](#6-kernel-and-system-info)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

Linux organises everything under one single tree starting at `/`. No C: or D: drives — every file, disk, device, and config lives somewhere beneath that root slash. The **shell** is the translator between you and the kernel — when you type a command, the shell interprets it and asks the kernel to do the work. On servers you interact with Linux through a shell over SSH with no GUI. These commands are how you see, move, and understand everything on that machine. They are the vocabulary you use in every other file in these notes.

---

## How it fits the stack

```
  L6  You  ← this file lives here
       ~ · /home/akhil · pwd · ls · whoami
  L5  Tools & Files
  L4  Config
  L3  State & Debug
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

L6 is where you land every time you SSH in. Before you can edit a config at L4, read a log at L3, or check a service at L1 — you need to be able to move around at L6. This file teaches that movement.

---

## 1. Directory Navigation

The shell always operates inside some directory — called the **current working directory (CWD)**. When you SSH into a server you have no idea where you landed. The first command you run is `pwd` — it tells you exactly where you are before you touch anything.

**Absolute vs relative paths:**
- **Absolute** — starts from root: `/home/akhil/webstore` — works from anywhere
- **Relative** — starts from your CWD: if you are in `/home/akhil`, then `cd webstore` takes you to `/home/akhil/webstore`
- `..` — means parent directory. `cd ..` moves you up one level.
- `~` — shorthand for your home directory. The shell replaces it with `/home/akhil`.

| Command | Full form | What it does | When you reach for it |
|---|---|---|---|
| `pwd` | Print Working Directory | Print the full path of your current location | First command after SSHing into any server |
| `cd <dir>` | Change Directory | Move into a directory | `cd ~/webstore/logs` — navigate to a specific folder |
| `cd ..` | Change Directory up | Move up one directory level | Going from `~/webstore/logs` back to `~/webstore` |
| `cd ~` | Change Directory home | Jump to your home directory | Getting back to a known starting point fast |
| `cd -` | Change Directory previous | Jump back to the last directory you were in | Toggling between two directories during an incident |
| `mkdir <dir>` | Make Directory | Create a new directory | `mkdir ~/webstore/logs` — creating the logs folder |
| `mkdir -p <path>` | Make Directory --parents | Create nested directories in one shot, no error if they exist | `mkdir -p ~/webstore/{frontend,api,db,logs,config,backup}` — builds the full project structure in one command |
| `rmdir <dir>` | Remove Directory | Delete an empty directory | Removing a folder you created by mistake — fails silently if the directory has contents |
| `rm -rf <dir>` | Remove --recursive --force | Delete a directory and everything inside it, no confirmation | Wiping a directory completely — always run `ls <path>` first to confirm what you are about to delete |

> **`rm -rf` has no confirmation and no undo.** A wrong path on a production server means permanent data loss. Build this habit: always run `ls <path>` first to confirm exactly what is there before deleting it.

---

## 2. Listing Directory Contents

`ls` is the command you run more than any other on a Linux server. Flags give you permissions, ownership, size, timestamps, and hidden files — all critical during incident work.

| Command | Full form | What it shows | When you reach for it |
|---|---|---|---|
| `ls` | List | Filenames only | Quick glance at what is in a directory |
| `ls -l` | List --long | Full details — permissions, owner, size, timestamp | Checking who owns the webstore config and when it was last changed |
| `ls -lh` | List --long --human-readable | Same as `-l` but sizes in KB/MB/GB | Checking whether a log file has silently grown to 2 GB overnight |
| `ls -la` | List --long --all | Full details including hidden files (`.` prefix) | Finding `.env` or `.bashrc` files invisible by default |
| `ls -lt` | List --long --time | Sorted by modification time, newest first | Spotting which file in `~/webstore/logs` changed during an incident |
| `ls -ltr` | List --long --time --reverse | Sorted by modification time, oldest first | Seeing the full chronological history of changes in a directory |
| `ls -ld <dir>` | List --long --directory | Info about the directory itself, not its contents | Checking permissions on `~/webstore/` without listing everything inside |

You can chain flags freely — `ls -lh`, `ls -ltr`, `ls -lahtr` all work. Flag order does not matter.

**Gold standard: `ls -lahtr`** — long format, all files including hidden, human-readable sizes, sorted oldest-to-newest. Full picture at a glance.

**Reading `ls -lh` output — every field explained:**

```bash
ls -lh ~/webstore/config/
# -rw-r--r-- 1 akhil www-data 1.2K Apr 5 09:14 webstore.conf
```

| Field | Value | What it means |
|---|---|---|
| File type + permissions | `-rw-r--r--` | `-` = regular file. Owner can read+write. Group and others can only read. |
| Hard links | `1` | One reference to this file in the filesystem |
| Owner | `akhil` | The user who owns this file |
| Group | `www-data` | Any process running as `www-data` — such as nginx — inherits the group's read permission |
| Size | `1.2K` | Human-readable because of `-h` — without it this shows raw bytes |
| Last modified | `Apr 5 09:14` | When the file was last written to — critical during incident triage |
| Filename | `webstore.conf` | The file |

When you see `www-data` as the group on a config file, it means nginx can read it — intentional. If the group were `root`, nginx would be locked out and the site would fail to start.

---

## 3. Terminal Essentials

The shell keeps a numbered history of every command you have run. On a server this lets you repeat long commands without retyping and audit what was run before you arrived.

| Command | What it does | When you reach for it |
|---|---|---|
| `clear` | Clear the terminal screen — history is untouched | Cleaning up visual clutter before a focused task |
| `history` | Show all commands run this session with line numbers | Auditing what was run on a server before you got there |
| `!<num>` | Re-run the command at that history number | `!42` — repeat a long command without retyping it |
| `!!` | Re-run the last command | Running the same command twice — or `sudo !!` to re-run with sudo |

**Keyboard shortcuts that save the most time:**

| Shortcut | What it does |
|---|---|
| `↑` | Scroll back through history one command at a time |
| `Ctrl + R` | Reverse-search history by typing part of a command |
| `Ctrl + C` | Kill the running command immediately |
| `Ctrl + L` | Clear the screen — same as `clear` |
| `Tab` | Autocomplete a command, filename, or path |

`Ctrl + R` is the one most people do not know but use constantly once they do. Type `Ctrl + R` then start typing `docker run` — the shell finds the last matching command. Press Enter to run it or keep typing to narrow the search.

---

## 4. System Information

These are the first commands you run when you SSH into an unfamiliar server. They tell you who you are, what the machine is, and whether anything unusual is happening.

| Command | What it tells you | When you reach for it |
|---|---|---|
| `whoami` | Your current username | Confirming you are logged in as the right user — not root when you shouldn't be |
| `id` | Your UID, GID, and all group memberships | Confirming you have the group access you need before touching files |
| `who` | Every user currently logged into this machine | Checking if someone else is on the server during an incident |
| `uptime` | How long the system has been running + load averages | A server that rebooted 3 minutes ago when it should have been up 30 days tells you something broke |
| `date` | Current system date and time | Confirming the server clock is correct before reading log timestamps |
| `hostname` | The machine's name | Confirming you are on the right server — critical when managing multiple machines |

**What `uptime` output actually means:**

```bash
uptime
# 10:32:11 up 4 days, 2:17, 1 user, load average: 0.45, 0.38, 0.31
```

The three load average numbers are CPU demand over the last 1 minute, 5 minutes, and 15 minutes. A number below your CPU core count means the system is healthy. A number above it means the system is under more load than it can handle.

```bash
# See your CPU core count to compare against load average
nproc
# 2   ← load average above 2.0 means this server is overloaded
```

---

## 5. Getting Help

Every command ships with documentation built in. Before searching the internet, check locally — it is faster and works on any server with no internet access.

| Command | What it does | When you reach for it |
|---|---|---|
| `man <command>` | Full manual page — everything the command can do | `man ls` — when you need to find an obscure flag |
| `<command> --help` | Short usage summary — faster than man | `cp --help` — quick reminder of the most common flags |
| `whatis <command>` | One-line description of what a command does | Quick reminder of what a command is for |
| `which <command>` | Shows the exact path of the executable that would run | `which python3` — confirming which Python is active when you have multiple versions installed |
| `whereis <command>` | Finds the binary, man page, and source locations | Confirming where a tool is installed on the system |

Inside `man` pages: use `/` to search, `n` to jump to the next match, `q` to quit. Most man pages are long — searching is faster than scrolling.

---

## 6. Kernel and System Info

`uname` (Unix Name) reports information about the running kernel and hardware. You reach for it when confirming the kernel version after an update, when a tool requires a specific architecture, or when a script needs to detect what OS it is running on.

| Command | Full form | What it shows | Example output |
|---|---|---|---|
| `uname -s` | uname --kernel-name | Kernel name | `Linux` |
| `uname -r` | uname --kernel-release | Kernel version | `6.5.0-26-generic` |
| `uname -n` | uname --nodename | Hostname | `webstore-prod-01` |
| `uname -m` | uname --machine | Hardware architecture | `x86_64` |
| `uname -a` | uname --all | All of the above in one line | Full system summary |

```bash
uname -a
# Linux webstore-prod-01 6.5.0-26-generic #26~22.04.1-Ubuntu SMP x86_64 GNU/Linux
```

---

## On the webstore

This is the first real work. You are on a blank server.
Before installing or deploying anything, you need to understand what you have.

```bash
# Step 1 — confirm who you are and where you landed
whoami
# akhil

pwd
# /home/akhil

# Step 2 — check the machine is healthy before touching anything
uptime
# 10:32:11 up 4 days, 2:17, 1 user, load average: 0.08, 0.03, 0.01

hostname
# webstore-prod-01

# Step 3 — check disk space — enough room to install software and store logs?
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        20G  4.2G   15G  23% /

# Step 4 — check memory
free -h
#               total        used        free
# Mem:           3.8G        1.1G        2.7G

# Step 5 — build the webstore directory structure
mkdir -p ~/webstore/{frontend,api,db,logs,config,backup}

# Step 6 — verify it was created correctly
ls -la ~/webstore/
# drwxrwxr-x 8 akhil akhil 4.0K Apr  5 09:14 .
# drwxr-xr-x 5 akhil akhil 4.0K Apr  5 09:14 ..
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 api
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 backup
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 config
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 db
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 frontend
# drwxrwxr-x 2 akhil akhil 4.0K Apr  5 09:14 logs

# Step 7 — navigate into the project and confirm your location
cd ~/webstore
pwd
# /home/akhil/webstore
```

The webstore directory structure now exists on disk. Every file from here onward operates inside this structure. The `frontend/` directory is where nginx will serve files from. The `logs/` directory is where access and error logs will land. The `config/` directory is where `webstore.conf` will live.

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `bash: cd: /path: No such file or directory` | The path does not exist or you have a typo | `ls` the parent directory first to confirm the exact name |
| `mkdir: cannot create directory: Permission denied` | You do not have write permission on the parent | `ls -ld <parent>` to check ownership — you may need `sudo` |
| `rm: cannot remove: Is a directory` | You used `rm` without `-r` on a directory | Use `rm -r` for directories — add `-f` only if you are sure |
| `rm -rf` deleted the wrong thing | Wrong path, no confirmation | No fix — no undo on Linux. Restore from backup. Verify paths before running. |
| `pwd` shows unexpected path | You navigated somewhere by accident | `cd ~` to get back to a known location, then navigate again |
| Load average in `uptime` is very high | System is overloaded | `top` or `htop` to find which process is consuming resources |

---

## Daily commands

| Command | What it does |
|---|---|
| `pwd` | Show where you are right now |
| `ls -lahtr` | List all files including hidden, human-readable sizes, sorted by time |
| `cd <path>` | Move into a directory |
| `cd ~` | Jump to your home directory |
| `cd -` | Jump back to the previous directory |
| `mkdir -p <path>` | Create nested directories in one command |
| `whoami` | Show your current username |
| `id` | Show your UID, GID, and all group memberships |
| `uptime` | Show how long the machine has been running and load average |

---

→ **Interview questions for this topic:** [99-interview-prep → Linux Basics](../99-interview-prep/README.md#linux-basics)
