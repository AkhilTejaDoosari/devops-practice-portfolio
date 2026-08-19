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

# Interview Prep — Linux

> Read the notes files first. Come here the day before an interview.   
> Each answer is 30 seconds. No more. That is what interviewers want.   

---

## Table of Contents

- [Boot Process](#boot-process)
- [Linux Basics](#linux-basics)
- [Working with Files](#working-with-files)
- [Filter Commands](#filter-commands)
- [sed](#sed)
- [awk](#awk)
- [vim](#vim)
- [Users and Groups](#users-and-groups)
- [Permissions](#permissions)
- [Archiving](#archiving)
- [Package Management](#package-management)
- [Service Management](#service-management)
- [Networking](#networking)
- [Logs and Debug](#logs-and-debug)

---

## Boot Process

**What happens when you press the power button on a Linux server?**

BIOS or UEFI firmware runs a POST check on the hardware, then finds the bootable disk and loads GRUB. GRUB loads the kernel and initramfs into RAM and hands off control. The kernel initialises hardware, uses initramfs to mount the real root filesystem, then starts systemd as PID 1. systemd reads unit files and brings all services up to the configured target — on servers that is `multi-user.target`. Then you get a login prompt.

**What is initramfs and why does it exist?**

initramfs is a tiny temporary filesystem loaded into RAM alongside the kernel. The kernel needs drivers to mount the real root filesystem, but those drivers might live on the real root filesystem — a chicken-and-egg problem. initramfs contains just enough drivers to break that deadlock. Once the real filesystem is mounted, initramfs is discarded.

**A server won't boot. You see `grub rescue>`. What happened and what do you do?**

GRUB failed to find the kernel. Common causes: disk UUID changed, wrong partition in grub.cfg, or the boot partition is corrupted. Boot from a live USB, chroot into the broken system, run `grub-install /dev/sda` and `update-grub` to regenerate the config.

**What is the difference between GRUB's two config files?**

`/etc/default/grub` is human-editable settings — timeout, default kernel, boot parameters. `/boot/grub/grub.cfg` is auto-generated from those settings. You never edit `grub.cfg` directly. You edit `/etc/default/grub` and run `sudo update-grub` to regenerate it.

**What is systemd and what is PID 1?**

systemd is the init system — the first process the kernel starts, always with PID 1. It is the parent of every other process on the system. It manages services through unit files, handles logging via journald, and manages the boot target. If systemd dies, the system goes down.

---

## Linux Basics

**What is the difference between `~` and `/`?**

`/` is the root of the entire filesystem — the top of the directory tree. Everything on the system lives somewhere beneath it. `~` is shorthand for the current user's home directory — `/home/akhil` for akhil. They are completely different locations.

**What does `ls -lahtr` show?**

Long format (`-l`) with all files including hidden (`-a`), sizes in human-readable KB/MB/GB (`-h`), sorted by modification time (`-t`) in reverse order oldest-first (`-r`). It gives you the full picture of a directory — permissions, owner, size, and when each file was last changed.

**What is the difference between an absolute and relative path?**

An absolute path starts from root and works from anywhere — `/home/akhil/webstore/config/webstore.conf`. A relative path starts from your current working directory — if you are in `/home/akhil`, then `webstore/config/webstore.conf` refers to the same file. `pwd` tells you your current directory.

**What does `uptime` tell you?**

How long the system has been running and the load average over the last 1, 5, and 15 minutes. A load average above the number of CPU cores means the system is under more load than it can handle. A very short uptime when you expect days or weeks tells you the server restarted unexpectedly.

---

## Working with Files

**What is the difference between `>` and `>>`?**

`>` redirects output to a file and overwrites it completely — no warning, no undo. `>>` appends to the end of the file without touching existing content. Using `>` when you meant `>>` is one of the most common causes of data loss on a server.

**What flags do you always use with `cp` when backing up a directory?**

`cp -riv` — `-r` to copy directories recursively, `-i` to prompt before overwriting so you cannot silently destroy an existing backup, `-v` to print each file as it copies so you can confirm what moved and catch wrong paths. This is the gold standard before any production change.

**What is a symlink and how does nginx use them?**

A symlink is a pointer to another file or directory. nginx keeps all site configs in `sites-available/` and enables them by creating symlinks in `sites-enabled/`. To disable a site you remove the symlink — the actual config file is untouched. To enable a site you create the symlink. The actual config is never moved or deleted.

**What is the difference between `stat` and `ls -l`?**

`ls -l` gives a readable summary — permissions, owner, size, last modified time. `stat` gives full metadata — all three timestamps (access, modify, change), exact size in bytes, inode number, and permissions in both octal and symbolic form. During incident triage, `stat` tells you the exact second a config file was last changed.

---

## Filter Commands

**What does `grep -v '200'` do and when do you use it?**

`-v` is invert-match — it shows every line that does NOT match the pattern. `grep -v '200'` on an access log shows every non-200 request — immediately surfaces all errors, 404s, 500s, and redirects in one command. It is the fastest way to see every problem in a log without scrolling.

**Explain the pipeline: `cut -d' ' -f1 access.log | sort | uniq -c | sort -rn`**

`cut` extracts field 1 (the IP address) from each line. `sort` puts identical IPs adjacent to each other. `uniq -c` counts and deduplicates consecutive identical lines. `sort -rn` sorts numerically in descending order. Result: a ranked list of which IPs are hitting the server most — the core pattern for log analysis.

**What is the difference between `find` and `locate`?**

`find` walks the live filesystem in real time — results are always current. It can filter by name, type, size, age, owner, and execute commands on matches. `locate` searches a prebuilt database — it is instant but only as fresh as the last time `updatedb` ran, usually daily. Use `find` when you need current results, `locate` when you just need to know where a file is and freshness does not matter.

**What does `tee` do?**

`tee` reads from stdin and writes to both stdout and a file at the same time. You see the output on screen and it gets saved — without running the command twice. Useful during an incident when you want to watch grep results live and save them for the team simultaneously.

---

## sed

**What does `sed -i 's/production/staging/' webstore.conf` do?**

`s/production/staging/` is the substitution command — find `production`, replace with `staging`, on the first match per line. `-i` writes the change back to the file in-place. Without `-i`, sed only prints the result to terminal and the file is untouched. This is the command a deploy script uses to switch config environments without opening an editor.

**What is the difference between `s/old/new/` and `s/old/new/g`?**

Without `g`, sed replaces only the first match per line. With `g` (global), it replaces every match on every line. If `old` appears three times on one line and you use `s/old/new/` without `g`, only the first occurrence changes.

**How do you replace a path like `/api/v1` with `/api/v2` in sed when the pattern contains forward slashes?**

Switch the delimiter. The default is `/` but sed accepts any character. Use `#` instead: `sed 's#/api/v1#/api/v2#g' file`. Any character that does not appear in your pattern or replacement works — `|` and `@` are also common.

**How do you preview what sed would change without actually changing the file?**

Run without `-i` — sed prints to terminal, file unchanged. Add `-n` with the `p` flag to see only the lines that would be changed: `sed -n 's/old/new/p' file`. This shows exactly which lines match before you commit to in-place editing.

---

## awk

**What is the basic structure of an awk command?**

`awk 'PATTERN { ACTION }' file`. awk reads the file one line at a time. For each line it checks if the pattern matches — if it does, it runs the action. Fields are split by whitespace by default, numbered from `$1`. `$0` is the whole line. `NR` is the current line number.

**What is the difference between `/500/` and `$4 == "500"` in awk?**

`/500/` matches any line containing `500` anywhere — in a URL path, an error message, anywhere. `$4 == "500"` matches only when field 4 is exactly the string `500`. Use field matching when you know which column holds the value you care about — it prevents false matches.

**How do you sum a column in awk?**

`awk '{ total += $5 } END { print total }' file`. Variables in awk persist across lines. `total += $5` adds field 5 of every line to `total`. `END` runs once after all lines are processed, printing the accumulated sum. The same pattern works for counts (`count++`) and averages (`total/NR`).

**What is an associative array in awk and what is it used for?**

An associative array uses a string as the key instead of a number. `count[$1]++` increments the count for each unique value of field 1. After processing the entire file, `for (ip in count) print count[ip], ip` prints the count for every unique IP. This is how you count occurrences per unique value in one pass.

---

## vim

**Someone opens vim for the first time and starts typing but nothing appears correctly. What is happening?**

vim starts in Normal mode — keys are commands, not text. They need to press `i` to enter Insert mode first, then type. When done, press `Esc` to return to Normal mode. `Esc` always returns you to Normal mode from anywhere — it is the first thing to try when vim behaves unexpectedly.

**How do you save and exit vim? How do you exit without saving?**

`:wq` saves and exits — `:w` writes the file, `:q` quits. `:q!` exits without saving and without asking — the `!` forces quit even with unsaved changes. If you opened the wrong file and made no intentional changes, `:q!` is the fastest exit.

**How do you jump to line 47 in vim?**

Type `47G` in Normal mode. `G` by itself jumps to the last line. `gg` jumps to the first line. `5G` jumps to line 5. This is how you go directly to a syntax error reported on a specific line.

**How do you replace all occurrences of a word in vim?**

`:%s/old/new/g` from Command-line mode (press `:` first). `%` applies to the entire file, `s` is substitute, `g` is global (all occurrences per line). `:%s/old/new/gc` adds a confirmation prompt for each match — useful when you want to review each change before applying it.

---

## Users and Groups

**What is the difference between UID and username?**

Linux tracks users by UID — a number. The username is just a human-readable label that resolves to the UID. When you see `akhil` as file owner in `ls -l`, Linux is storing UID 1000 and your terminal is resolving it. This matters when moving files between servers — if UIDs differ, ownership mappings break.

**What is the danger of running `usermod -G webstore-team akhil` without `-a`?**

Without `-a` (append), `-G` replaces all of the user's current group memberships with only the groups you specified. Akhil loses membership in sudo, docker, and every other group instantly. They lose all access those groups granted. Always use `-aG` to add a group without touching existing memberships.

**What is the UID range for system accounts and why does it matter?**

UIDs 1-999 are reserved for system accounts — daemons and services like `www-data` (33) and `postgres` (999). They have no login shell and no home directory by design. Human users get UID 1000+. This separation ensures services and humans never share an identity, which limits blast radius if a service is compromised.

**Why should nginx not run as root?**

If nginx runs as root and is compromised, the attacker has full system access — can read any file, write anywhere, execute anything. Running nginx as `www-data` with access only to the files it needs means a compromise is contained — the attacker gets www-data's limited permissions, not root's unlimited ones. This is the principle of least privilege.

---

## Permissions

**What does `chmod 640` mean?**

Six = owner gets read+write (4+2). Four = group gets read only (4). Zero = others get nothing (0). In symbolic form: `-rw-r-----`. Used on config files containing secrets — the owner can edit them, the service group can read them, everyone else is locked out.

**What is the execute bit on a directory and why does it matter?**

Without execute (`x`) on a directory, you cannot `cd` into it or access any files inside — even if you have read permission. Read without execute lets you see filenames with `ls` but nothing else. This is a common trap — a directory with `r--` looks accessible but is effectively locked.

**What is SGID on a directory and when do you use it?**

SGID (Set Group ID) on a directory makes all new files created inside automatically inherit the directory's group instead of the creator's primary group. Essential for shared team directories — every file created in `~/webstore/logs/` belongs to `webstore-team` regardless of who created it. Set with `chmod g+s <dir>`.

**Why use `640` instead of `644` for a config file with a database password?**

`644` means others (any user on the server) can read the file — including the database password. `640` means only the owner and the group can read it. nginx runs as `www-data` which is a member of `webstore-team`, so it can read the config. Any other user on the server cannot.

---

## Archiving

**What is the difference between archiving and compression?**

Archiving combines multiple files into one file — no size reduction, the purpose is portability. `tar` archives. Compression reduces a file's size. `gzip` compresses. `tar.gz` does both — archive first with `tar`, then compress the result with `gzip`. On Linux servers you almost always use `tar.gz`.

**What does `tar -czvf backup.tar.gz ~/webstore/` do, flag by flag?**

`c` = create a new archive. `z` = compress it with gzip. `v` = verbose, print each file. `f` = the next argument is the filename. `backup.tar.gz` = the output file. `~/webstore/` = what to archive. Reading it: create a gzip-compressed verbose archive called backup.tar.gz from the webstore directory.

**Why do you always list the archive contents before extracting?**

`tar -tzvf archive.tar.gz` shows every file without extracting. It confirms the archive contains what you expect and that the paths are correct. Extracting blindly into the wrong directory can overwrite existing files — listing first costs 2 seconds and prevents that mistake.

**Why use `tar.gz` instead of `zip` on Linux servers?**

`tar.gz` preserves Unix file permissions, ownership, and symlinks exactly. `zip` does not reliably preserve Unix permissions. If you archive the webstore with `zip` and extract it on another Linux server, the file permissions will be wrong and you will have to run `chown` and `chmod` again to fix them.

---

## Package Management

**What is the difference between `apt update` and `apt upgrade`?**

`apt update` refreshes the local package index — it fetches the latest list of available versions from the repositories but installs nothing. `apt upgrade` installs newer versions of all packages already installed. You must run `update` before `upgrade` so apt knows what versions are available.

**What is the difference between `apt remove` and `apt purge`?**

`apt remove` uninstalls the package binaries but leaves config files on disk. If you reinstall later, your config is still there. `apt purge` removes everything including config files — a clean slate. Use `remove` when you plan to reinstall. Use `purge` for a complete uninstall.

**Why do you never skip `apt update` before `apt install`?**

Without updating the index, apt installs from its cached list which may be days or weeks old. You might install an outdated version with known security issues, or apt might fail entirely because a dependency was renamed and the old name no longer exists in the current repository.

**What does `apt install nginx` actually do to the system?**

Downloads the nginx package and all its dependencies. Installs the nginx binary to `/usr/sbin/nginx`. Puts the default config in `/etc/nginx/`. Creates the `www-data` system user if it does not exist. Registers `nginx.service` with systemd. On Ubuntu, nginx starts automatically after install.

---

## Service Management

**What is the difference between `systemctl start` and `systemctl enable`?**

`start` runs the service right now but has no effect on boot behavior. `enable` makes the service start automatically on the next boot but does not start it now. In production you almost always want both — `systemctl enable --now nginx` enables and starts in one command.

**What is the difference between `systemctl restart` and `systemctl reload`?**

`restart` stops the process completely and starts a fresh one — all active connections are dropped. `reload` sends a signal asking the process to re-read its config without stopping — active connections are preserved. Always use `reload` for config changes in production when the service supports it. Test config with `nginx -t` before either.

**A service fails to start. What is your debugging sequence?**

`systemctl status <service>` to see the current state and the last few log lines. `journalctl -u <service> -n 50` to read the full error. Fix what the log identifies. `nginx -t` if it is nginx to confirm the config is valid. `systemctl start <service>` again.

**What are the three unit file priority locations and which wins?**

1. `/etc/systemd/system/` — your overrides, highest priority, always wins
2. `/run/systemd/system/` — runtime units, second priority, lost on reboot
3. `/usr/lib/systemd/system/` — vendor defaults installed by packages, lowest priority, never edit these

---

## Networking

**What does `ss -tlnp` show?**

All listening TCP ports with the process name and PID that owns each one. `-t` = TCP, `-l` = listening, `-n` = numeric (no DNS resolution), `-p` = process name. After deploying a service, `ss -tlnp` confirms it is bound to the expected port. If it is not in the list, it is not running or failed to bind.

**What is the difference between a service bound to `0.0.0.0` vs `127.0.0.1`?**

`0.0.0.0` means the service accepts connections on all network interfaces — accessible from outside the server. `127.0.0.1` (loopback) means the service only accepts connections from the same machine — not accessible from outside. The database should always be on `127.0.0.1`. nginx serving the web should be on `0.0.0.0`.

**The API cannot connect to the database. How do you isolate the problem?**

`nc -zv webstore-db 5432` from the API server. If nc succeeds, the network is fine — the problem is in the application layer (wrong password, wrong database name, wrong connection string). If nc fails, it is a network problem — check if postgres is running, check firewall rules, check if postgres is listening on `127.0.0.1` instead of `0.0.0.0`.

**What does `dig +short webstore-api.example.com` return and when do you use it?**

Just the IP address the hostname resolves to. Use it after a DNS change to confirm the record has propagated, or when traffic is hitting the wrong server — `dig +short` immediately tells you if DNS is returning the old IP. `dig @8.8.8.8 hostname` queries Google's DNS directly to bypass your local cache.

**What does `502 Bad Gateway` mean?**

nginx received the request and forwarded it to the upstream service (the API), but the upstream did not respond. nginx is running. The API is not. Check `systemctl status <api-service>` and `journalctl -u <api-service> -n 50`.

---

## Logs and Debug

**A server is having issues and you SSH in. What are your first three commands?**

`uptime` — is the machine alive, how long has it been up, is load normal. `df -h` — is any filesystem full (full disk causes silent failures). `systemctl list-units --state=failed` — are there any failed services. These three commands tell you the state of the system in 10 seconds.

**Where do you look first when a service fails to start?**

`journalctl -u <service> -n 50` — the last 50 lines of the service's log since the failed start. systemd captures all service output here. The exact error message is almost always in those 50 lines. `systemctl status <service>` shows a shorter summary but `journalctl` gives more context.

**Disk is filling up on a server. How do you find what is consuming it?**

`df -h` shows which filesystem is full. `du -sh /var/log/* | sort -rh | head -10` shows the largest items in `/var/log`. `du -sh /* | sort -rh | head -10` shows the largest top-level directories. Nine times out of ten on a server, it is a log file that was not rotated.

**What is the difference between `/var/log` and `/run`?**

`/var/log` is persistent — it survives reboots and accumulates over time. It contains the historical record of what happened. `/run` is a RAM filesystem — it exists only while the system is running and is wiped completely on every reboot. It contains live state: socket files, PID files, session data. Both are at L3 but serve different purposes.

**What does `dmesg | grep -i oom` tell you?**

OOM stands for Out of Memory. When the Linux kernel runs out of RAM, it activates the OOM killer — it selects and kills processes to free memory. `dmesg | grep -i oom` shows if this happened and which processes were killed. A process that mysteriously disappeared without a trace was probably OOM killed.
