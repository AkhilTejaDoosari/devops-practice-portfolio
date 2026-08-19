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

# Service Management

> **Layer:** L1 — Process Manager   
> **Depends on:** [11 Package Management](../11-package-management/README.md) — you need to install software before you can manage it as a service   
> **Used in production when:** Starting and stopping services, making a service survive reboots, applying a config change without dropping connections, or debugging why a service failed to start   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. Services and daemons](#1-services-and-daemons)
- [2. systemd — how it manages services](#2-systemd--how-it-manages-services)
- [3. systemctl — the control interface](#3-systemctl--the-control-interface)
- [4. restart vs reload — the critical distinction](#4-restart-vs-reload--the-critical-distinction)
- [5. journalctl — reading service logs](#5-journalctl--reading-service-logs)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

A service is a process that runs in the background without user interaction — started at boot, running continuously, doing its job silently until something goes wrong. nginx serving the webstore frontend is a service. The SSH daemon that lets you log in remotely is a service. On modern Linux, all of these are managed by `systemd` — the PID 1 process from file 01. Every service you start, stop, enable, or debug goes through `systemctl`, systemd's command-line interface.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files
  L4  Config  ← /etc/systemd/system/ — your unit files live here
  L3  State & Debug  ← /var/log/ /run/ — service logs and state live here
  L2  Networking
  L1  Process Manager  ← this file lives here
       systemd PID 1 · systemctl · journalctl
  L0  Kernel & Hardware
```

Every service you install at L5 (apt install nginx) gets managed by systemd at L1. The config at L4 (/etc/nginx/) controls what the service does. The logs at L3 (/var/log/nginx/) record what it did.

---

## 1. Services and daemons

A **daemon** is a background process that keeps running until the system shuts down. Every daemon has a config file:

| Daemon | What it does | Config file |
|---|---|---|
| `nginx` | Serves web content | `/etc/nginx/nginx.conf` |
| `sshd` | Accepts SSH connections | `/etc/ssh/sshd_config` |
| `cron` | Runs scheduled tasks | `/etc/crontab`, `/etc/cron.d/` |
| `journald` | Collects all system logs | `/etc/systemd/journald.conf` |
| `postgresql` | Runs the database | `/etc/postgresql/*/main/postgresql.conf` |

When you edit a config file, nothing changes until you tell the service to reload or restart. The running process in memory uses the old config until you explicitly apply the new one.

---

## 2. systemd — how it manages services

systemd manages services through **unit files** — text files describing what binary to run, what user to run it as, what it depends on, and whether it should restart if it crashes.

**Unit file locations — priority order (1 wins):**

| Priority | Location | Purpose |
|---|---|---|
| 1 | `/etc/systemd/system/` | Your overrides and custom units — edit here |
| 2 | `/run/systemd/system/` | Runtime units — lost on reboot |
| 3 | `/usr/lib/systemd/system/` | Vendor defaults installed by packages — never edit |

**Unit types:**

| Extension | Purpose |
|---|---|
| `.service` | Background daemon — nginx, sshd, postgresql |
| `.timer` | Scheduled job — modern cron replacement |
| `.socket` | Socket-activated service |
| `.target` | Group of units — defines boot state |

---

## 3. systemctl — the control interface

**Start, stop, restart, reload:**

```bash
# Start now — does not affect boot behavior
sudo systemctl start nginx

# Stop now
sudo systemctl stop nginx

# Restart — stop then start — drops all active connections
sudo systemctl restart nginx

# Reload — apply new config without dropping connections
sudo systemctl reload nginx
```

**Enable and disable at boot:**

```bash
# Enable — will start automatically on next boot
sudo systemctl enable nginx

# Enable AND start immediately in one command
sudo systemctl enable --now nginx

# Disable — will not start on boot
sudo systemctl disable nginx

# Disable AND stop immediately
sudo systemctl disable --now nginx
```

`enable` and `start` are independent. `enable` without `start` = starts next boot but not now. `start` without `enable` = running now but not after reboot. In production you almost always want both — use `enable --now`.

**Check status:**

```bash
sudo systemctl status nginx
# ● nginx.service - A high performance web server
#      Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
#      Active: active (running) since Sat 2025-04-05 09:14:22 UTC; 2h ago
#     Main PID: 1235 (nginx)
#      CGroup: /system.slice/nginx.service
#              ├─1235 nginx: master process
#              └─1236 nginx: worker process

# Quick checks
systemctl is-active nginx     # active  or  inactive
systemctl is-enabled nginx    # enabled  or  disabled
```

**Loaded** = unit file found and whether it is enabled.
**Active** = current running state and how long.
**CGroup** = every process this service spawned.

**List services:**

```bash
systemctl list-units --type=service --state=running   # all running
systemctl list-units --type=service --state=failed    # all failed — check this first
```

---

## 4. restart vs reload — the critical distinction

**`restart`** — stops the process completely, starts a fresh one. Any user currently connected loses their connection. Use when a config change requires a full restart, or when a service is misbehaving.

**`reload`** — sends a signal asking the process to re-read its config. Process stays running. Connections are not dropped. nginx supports reload — new workers start with new config while old workers finish current requests, then exit gracefully.

```bash
# Edited nginx.conf — test first, then reload
sudo nginx -t                    # always test config syntax before applying
# nginx: configuration file /etc/nginx/nginx.conf test is successful
sudo systemctl reload nginx      # apply without dropping connections

# nginx consuming memory and not responding — restart it
sudo systemctl restart nginx     # drops connections, starts fresh
```

**The rule:** for config changes on a running production server, always try `reload` first. Only use `restart` when `reload` is not supported or when the service needs to be killed.

---

## 5. journalctl — reading service logs

systemd collects all service output in a centralized journal. `journalctl` is how you read it.

```bash
# All logs for nginx
journalctl -u nginx

# Follow live — new lines appear as written
journalctl -u nginx -f

# Last 50 lines
journalctl -u nginx -n 50

# Logs since current boot
journalctl -u nginx -b

# Logs from last hour
journalctl -u nginx --since "1 hour ago"

# Error-level messages only
journalctl -u nginx -p err

# Logs from previous boot — useful after a crash
journalctl -u nginx -b -1
```

**The debug loop when a service fails:**

```bash
sudo systemctl start nginx          # attempt to start
sudo systemctl status nginx         # see if it started or shows error
journalctl -u nginx -n 50           # read exactly what went wrong
# fix the problem
sudo nginx -t                       # verify config is valid
sudo systemctl start nginx          # try again
```

---

## On the webstore

The complete nginx lifecycle — from install to serving the webstore frontend.

```bash
# Step 1 — install nginx (from file 11)
sudo apt update && sudo apt install -y nginx

# Step 2 — nginx auto-starts on Ubuntu after install — check it
sudo systemctl status nginx
# Active: active (running)

# Step 3 — create webstore frontend directory and page
sudo mkdir -p /var/www/webstore-frontend
echo "<h1>webstore is live</h1>" | sudo tee /var/www/webstore-frontend/index.html

# Step 4 — write the nginx site config using vim (from file 07)
sudo vim /etc/nginx/sites-available/webstore
# [write the server block config]

# Step 5 — enable the site
sudo ln -s /etc/nginx/sites-available/webstore /etc/nginx/sites-enabled/webstore
sudo rm /etc/nginx/sites-enabled/default

# Step 6 — ALWAYS test config before applying
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Step 7 — reload without dropping connections
sudo systemctl reload nginx

# Step 8 — verify it is serving the webstore
curl http://localhost
# <h1>webstore is live</h1>

# Step 9 — enable nginx to survive reboots
sudo systemctl enable nginx
# Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service

# Step 10 — confirm enabled
systemctl is-enabled nginx
# enabled
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Job for nginx.service failed` | Config syntax error or port conflict | `sudo nginx -t` to find syntax error · `ss -tlnp \| grep :80` to find port conflict |
| Service starts but stops immediately | Binary error, missing file, or permission problem | `journalctl -u nginx -n 50` — read the exact error |
| `reload` returns error | Service does not support reload | Use `restart` instead — check man page for support |
| Service running but not surviving reboot | Started with `start` but not `enable` | `sudo systemctl enable nginx` |
| `enable` but service not starting after reboot | Unit file has dependency issue | `journalctl -b -u nginx` — logs from boot |
| Config change not taking effect | Forgot to reload after editing | `sudo systemctl reload nginx` or `restart` |

---

## Daily commands

| Command | What it does |
|---|---|
| `sudo systemctl status <svc>` | Full status — state, PID, recent logs |
| `sudo systemctl start <svc>` | Start service now |
| `sudo systemctl stop <svc>` | Stop service now |
| `sudo systemctl restart <svc>` | Stop and start — drops connections |
| `sudo systemctl reload <svc>` | Apply new config — no dropped connections |
| `sudo systemctl enable --now <svc>` | Enable at boot AND start immediately |
| `systemctl list-units --state=failed` | Show every service that failed |
| `journalctl -u <svc> -f` | Follow live service logs |
| `journalctl -u <svc> -n 50` | Last 50 lines of service logs |
| `sudo nginx -t` | Test nginx config syntax before applying |

---

→ **Interview questions for this topic:** [99-interview-prep → Service Management](../99-interview-prep/README.md#service-management)
