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

# Package Management

> **Layer:** L5 — Tools & Files   
> **Depends on:** [02 Basics](../02-basics/README.md) — you need basic navigation before installing software   
> **Used in production when:** Installing nginx, updating the server, removing a package cleanly, or auditing what is installed on an unfamiliar server   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. What a package manager does](#1-what-a-package-manager-does)
- [2. APT — Debian and Ubuntu](#2-apt--debian-and-ubuntu)
- [3. YUM and DNF — RHEL CentOS Fedora](#3-yum-and-dnf--rhel-centos-fedora)
- [4. Comparing package managers](#4-comparing-package-managers)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

On a Linux server you never download software from a website and run an installer. You use the package manager — a tool that fetches verified software from trusted repositories, resolves all dependencies automatically, and tracks everything it installed so it can be cleanly removed later. This is how nginx gets on the webstore server. One command. No manual download. No guessing which libraries it needs. The package manager handles all of it.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       apt yum dnf — install and manage all software on the system
  L4  Config  ← /etc/apt/sources.list — where apt looks for packages
  L3  State & Debug  ← /var/cache/apt/ /var/lib/dpkg/ — package state lives here
  L2  Networking
  L1  Process Manager  ← systemd unit files created by packages land at L1
  L0  Kernel & Hardware
```

When you run `apt install nginx`, it downloads the package, installs the binary to `/usr/bin/`, puts the config in `/etc/nginx/`, and registers `nginx.service` with systemd at L1. One command touches four layers.

---

## 1. What a package manager does

A **package** is a bundle containing everything a piece of software needs — the binary, its libraries, default config files, and documentation. The package manager handles four things you would otherwise do manually:

- **Installation** — downloads the package and puts every file in the right place
- **Dependency resolution** — figures out what other packages this one needs and installs those too
- **Verification** — checks GPG signatures to confirm the package has not been tampered with
- **Removal** — tracks every file installed so it can cleanly remove them later

**Two ecosystems on Linux:**

| Ecosystem | Format | Manager | Used on |
|---|---|---|---|
| Debian | `.deb` | `apt` | Ubuntu, Debian — what this runbook uses |
| Red Hat | `.rpm` | `yum` / `dnf` | RHEL, CentOS, Fedora, Amazon Linux |

Ubuntu is the AWS EC2 default and what this runbook uses throughout. You will see both in real jobs.

---

## 2. APT — Debian and Ubuntu

APT (Advanced Package Tool) manages packages on Ubuntu. Package lists live in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/`. The index must be updated manually before installing.

**The install sequence — always in this order:**

```bash
# Step 1 — refresh the package index
# Does NOT install anything — just updates what apt knows is available
sudo apt update

# Step 2 — install
sudo apt install nginx
# apt resolves dependencies, downloads, installs in correct order
# creates www-data user, puts config in /etc/nginx/, registers systemd service
```

Never skip `apt update` before installing. Without it you may install a stale version, or apt may fail to find a dependency that was recently renamed.

**Full APT command set:**

| Command | What it does | When you reach for it |
|---|---|---|
| `sudo apt update` | Refresh package index — fetch latest available versions | Before every install or upgrade |
| `sudo apt install <pkg>` | Install a package and its dependencies | Installing nginx, curl, vim, git |
| `sudo apt install <pkg>=<version>` | Install a specific version | Pinning nginx to match production |
| `sudo apt upgrade -y` | Upgrade all installed packages | Routine server maintenance |
| `sudo apt remove <pkg>` | Remove package, keep config files | Removing nginx while keeping `/etc/nginx/` for reinstall |
| `sudo apt purge <pkg>` | Remove package AND all config files | Complete clean uninstall |
| `sudo apt autoremove` | Remove unused dependency packages | After removing a package that pulled in many deps |
| `sudo apt clean` | Delete downloaded `.deb` files from cache | Freeing disk space |
| `apt list --installed` | List all installed packages | Auditing what is on a server |
| `apt show <pkg>` | Show details — version, size, dependencies | Before installing, check what you are getting |
| `apt search <keyword>` | Search available packages | Finding the right package name |

**`remove` vs `purge` — when it matters:**
`apt remove nginx` removes the binary but leaves `/etc/nginx/` intact — if you reinstall later your config is still there.
`apt purge nginx` removes everything including configs — nothing left behind.
Use `remove` when you plan to reinstall. Use `purge` for a complete clean uninstall.

---

## 3. YUM and DNF — RHEL CentOS Fedora

YUM is the package manager on older Red Hat systems (RHEL 7, CentOS 7). DNF replaced it on RHEL 8+, Fedora, and Amazon Linux 2023.

**YUM (CentOS / RHEL 7):**

```bash
sudo yum install nginx       # install
sudo yum update -y           # upgrade all packages
sudo yum remove nginx        # remove
sudo yum clean all           # clear cached data
sudo yum list installed      # list installed packages
```

**DNF (Fedora / RHEL 8+ / Amazon Linux 2023):**

```bash
sudo dnf install nginx       # install
sudo dnf upgrade -y          # upgrade all packages
sudo dnf remove nginx        # remove
sudo dnf clean all           # clear cached data
sudo dnf list installed      # list installed packages
```

Key difference from APT: YUM and DNF do not separate `update` (refresh index) from `upgrade` (install updates). `yum update` and `dnf upgrade` do both in one step.

---

## 4. Comparing package managers

| | APT | YUM | DNF |
|---|---|---|---|
| Used on | Ubuntu, Debian | CentOS, RHEL 7 | Fedora, RHEL 8+, Amazon Linux |
| Format | `.deb` | `.rpm` | `.rpm` |
| Refresh index | `apt update` (manual) | Automatic | Automatic |
| Install | `apt install` | `yum install` | `dnf install` |
| Upgrade all | `apt upgrade` | `yum update` | `dnf upgrade` |
| Remove | `apt remove` | `yum remove` | `dnf remove` |
| Remove + configs | `apt purge` | No equivalent | No equivalent |
| Clean cache | `apt clean` | `yum clean all` | `dnf clean all` |
| List installed | `apt list --installed` | `yum list installed` | `dnf list installed` |
| Repo config | `/etc/apt/sources.list` | `/etc/yum.repos.d/` | `/etc/yum.repos.d/` |

---

## On the webstore

Installing the full webstore stack on a fresh Ubuntu server.

```bash
# Step 1 — update index first, always
sudo apt update

# Step 2 — install nginx to serve the webstore frontend
sudo apt install -y nginx
# -y (--yes) answers confirmation prompt automatically
# use -y in scripts; skip it interactively to review dependencies first

# Step 3 — confirm nginx installed and check version
nginx -v
# nginx version: nginx/1.24.0

# Step 4 — install tools needed for the webstore
sudo apt install -y curl vim git

# Step 5 — install PostgreSQL client to connect to webstore-db
sudo apt install -y postgresql-client

# Step 6 — verify what got installed
apt list --installed | grep -E 'nginx|curl|vim|git|postgresql'

# Step 7 — check disk space — installs can be large
df -h
# Filesystem      Size  Used Avail Use%
# /dev/sda1        20G  5.1G   14G  27%

# Step 8 — clean up downloaded .deb files
sudo apt clean
sudo apt autoremove
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `E: Unable to locate package <pkg>` | Package index is stale or package name is wrong | Run `sudo apt update` first, then `apt search <keyword>` to find correct name |
| `E: Could not get lock /var/lib/dpkg/lock` | Another apt process is running | Wait for it to finish, or `sudo kill <pid>` if it is stuck |
| `apt upgrade` broke a service | A package update changed behavior or config | Check `/var/log/dpkg.log` for what changed, restore from backup |
| `apt remove` left config files behind | Used `remove` instead of `purge` | Run `sudo apt purge <pkg>` to remove configs too |
| Disk full after install | `/var/cache/apt/archives/` filled with `.deb` files | `sudo apt clean` to clear the cache |
| Package installs old version | Index not refreshed before install | Always `sudo apt update` before `apt install` |

---

## Daily commands

| Command | What it does |
|---|---|
| `sudo apt update` | Refresh package index — run before every install |
| `sudo apt install -y <pkg>` | Install a package without confirmation prompt |
| `sudo apt upgrade -y` | Upgrade all installed packages |
| `sudo apt remove <pkg>` | Remove package, keep config files |
| `sudo apt purge <pkg>` | Remove package and all config files |
| `sudo apt autoremove` | Remove unused dependency packages |
| `sudo apt clean` | Clear downloaded .deb files from cache |
| `apt list --installed \| grep <name>` | Check if a specific package is installed |
| `apt show <pkg>` | Show package version, size, and dependencies |
| `apt search <keyword>` | Find the correct package name |

---

→ **Interview questions for this topic:** [99-interview-prep → Package Management](../99-interview-prep/README.md#package-management)
