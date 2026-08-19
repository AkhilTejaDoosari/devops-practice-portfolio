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

# Logs & Debug

> **Layer:** L3 — State & Debug   
> **Depends on:** [12 Service Management](../12-service-management/README.md) and [13 Networking](../13-networking/README.md) — logs only make sense once services are running and network is configured   
> **Used in production when:** Something broke. You SSH in at 2am. You need to find the cause using only the terminal.   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. Where logs live — /var/log](#1-where-logs-live--varlog)
- [2. journalctl — the systemd journal](#2-journalctl--the-systemd-journal)
- [3. /run — live runtime state](#3-run--live-runtime-state)
- [4. /var/lib — persistent app state](#4-varlib--persistent-app-state)
- [5. The debug workflow](#5-the-debug-workflow)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

L3 is where the system keeps its memory — what happened (logs), what is running now (runtime state), and what persists between reboots (app state). When something breaks in production, this is the first layer you open. Not the code. Not the config. The logs. The log tells you what actually happened, when it happened, and often exactly why. Every tool in files 01 through 13 produced information that landed here. This file teaches you how to read it.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files
  L4  Config  ← /etc controls what services do
  L3  State & Debug  ← this file lives here
       /var/log  /run  /var/lib  journalctl
  L2  Networking
  L1  Process Manager  ← systemd writes to journal at L3
  L0  Kernel & Hardware
```

Everything that happens at every other layer leaves a trace at L3. The kernel logs hardware events. systemd logs service starts and stops. nginx logs every request. When something breaks, L3 has the evidence.

---

## 1. Where logs live — /var/log

```
/var/log/
├── syslog          ← main system log — everything goes here
├── auth.log        ← all logins, sudo uses, SSH connections
├── kern.log        ← kernel messages — hardware errors, OOM kills
├── dpkg.log        ← every package install and remove
├── apt/
│   └── history.log ← apt install/upgrade/remove history
├── nginx/
│   ├── access.log  ← every HTTP request nginx received
│   └── error.log   ← nginx errors — config problems, upstream failures
└── journal/        ← systemd binary journal (read with journalctl)
```

**The log you open first depends on the symptom:**

| Symptom | First log to check |
|---|---|
| Service failed | `journalctl -u <service> -n 50` |
| Login or sudo issue | `/var/log/auth.log` |
| System slow or crashing | `/var/log/kern.log` or `dmesg` |
| nginx 502 or 503 | `/var/log/nginx/error.log` |
| Package install problem | `/var/log/dpkg.log` |
| General "something happened" | `/var/log/syslog` |

**Reading logs with the tools from file 04:**

```bash
# Watch syslog live
tail -f /var/log/syslog

# Find all errors in the last hour of syslog
grep "$(date +'%b %e %H')" /var/log/syslog | grep -i error

# Count nginx 500 errors today
grep "$(date +'%d/%b/%Y')" /var/log/nginx/access.log | grep ' 500 ' | wc -l

# Find which IPs are hitting the webstore with 500 errors
grep ' 500 ' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Check who has logged in recently
tail -n 50 /var/log/auth.log | grep 'Accepted'
```

---

## 2. journalctl — the systemd journal

`journalctl` reads the systemd binary journal — the central log for all services managed by systemd. More powerful than plain log files because it is structured, indexed, and searchable by time, priority, and service.

```bash
# All logs from current boot
journalctl -b

# All logs from previous boot — useful after a crash reboot
journalctl -b -1

# Logs for a specific service
journalctl -u nginx

# Follow live — new lines appear as they are written (-f = --follow)
journalctl -u nginx -f

# Last 50 lines of a service log (-n = --lines)
journalctl -u nginx -n 50

# Errors only — no info or debug noise (-p = --priority)
journalctl -u nginx -p err

# Logs from the last hour
journalctl -u nginx --since "1 hour ago"

# Logs between two timestamps
journalctl --since "2025-04-05 09:00" --until "2025-04-05 10:00"

# No pager — print all to terminal (useful in scripts)
journalctl -u nginx -n 100 --no-pager

# Show kernel messages (same as dmesg but via journal)
journalctl -k
```

**Priority levels** (lowest number = highest severity):
`0` = emerg · `1` = alert · `2` = crit · `3` = err · `4` = warning · `5` = notice · `6` = info · `7` = debug

`-p err` shows levels 0 through 3 — emergencies, alerts, critical, and errors.

---

## 3. /run — live runtime state

`/run` is a tmpfs (RAM filesystem) — it exists only while the system is running. Wiped completely on every reboot. Everything here reflects the current live state.

```
/run/
├── nginx.pid           ← PID of the running nginx master process
├── docker.sock         ← Docker API socket — CLI talks to daemon here
├── systemd/
│   └── system/         ← runtime unit files — temporary overrides
└── user/
    ├── 1000/           ← akhil's runtime session data
    ├── 1001/           ← charan's session
    ├── 1002/           ← pramod's session
    ├── 1003/           ← navya's session
    └── 1004/           ← indhu's session
```

```bash
# See what is in /run right now
ls /run/

# Find which PID nginx is running as
cat /run/nginx.pid
# 1235

# Confirm the PID is actually running
ps aux | grep 1235

# Check if docker socket exists (Docker is running)
ls -la /run/docker.sock
# srw-rw---- root docker  /run/docker.sock
```

---

## 4. /var/lib — persistent app state

`/var/lib` stores data that services write to disk and need to survive reboots — unlike `/run` which is wiped, `/var/lib` persists.

```
/var/lib/
├── docker/         ← all Docker images, containers, and volumes
├── dpkg/           ← installed package database — what apt knows is installed
├── apt/lists/      ← cached package lists from last apt update
├── mysql/          ← MySQL database files (if installed)
└── postgresql/     ← PostgreSQL database files (if installed)
```

```bash
# See disk usage of Docker data
du -sh /var/lib/docker/
# 8.2G /var/lib/docker/

# See disk usage of all /var/lib subdirectories
du -sh /var/lib/* | sort -rh | head -10

# Check the installed package database
ls /var/lib/dpkg/info/ | grep nginx
# nginx.list  nginx.md5sums  nginx.postinst  nginx.postrm  nginx.prerm
```

---

## 5. The debug workflow

This is the structured approach to any production problem. Work from symptom to cause in order.

```
SYMPTOM → LAYER → FIRST COMMAND → WHAT IT TELLS YOU
```

**Step 1 — is the machine alive?**
```bash
uptime          # is it up? how long? is load average normal?
free -h         # is RAM exhausted? (OOM killer may have struck)
df -h           # is disk full? (disk full = services fail silently)
```

**Step 2 — did anything fail at boot or recently?**
```bash
systemctl list-units --state=failed     # any failed units?
journalctl -b -p err                    # any errors since last boot?
dmesg | tail -20                        # any kernel errors?
```

**Step 3 — is the specific service running?**
```bash
systemctl status nginx
journalctl -u nginx -n 50
```

**Step 4 — is the service reachable on the network?**
```bash
sudo ss -tlnp | grep :80    # is nginx bound to the right port?
curl -I http://localhost     # does it respond to HTTP?
nc -zv localhost 80          # can you connect to the port at all?
```

**Step 5 — what do the application logs say?**
```bash
tail -f /var/log/nginx/error.log
grep ' 500 ' /var/log/nginx/access.log | tail -20
journalctl -u nginx -p err --since "1 hour ago"
```

**Step 6 — is disk or memory the problem?**
```bash
df -h                           # which filesystem is full?
du -sh /var/log/* | sort -rh    # which log is consuming the most space?
free -h                         # how much RAM is left?
dmesg | grep -i 'oom\|killed'   # has the OOM killer been running?
```

---

## On the webstore

A full incident simulation. Users report the webstore is returning 502 errors.

```bash
# Step 1 — is the machine alive?
uptime
# 14:23:11 up 12 days, load average: 0.45, 0.38, 0.31  ← machine is fine

df -h
# /dev/sda1  20G  19G  500M  98%  /   ← disk nearly full!

# Disk is nearly full — this could cause the issue
# Find what is eating space
du -sh /var/log/* | sort -rh | head -5
# 18G  /var/log/nginx/access.log   ← the culprit

# Step 2 — is nginx up?
systemctl status nginx
# Active: active (running)

# Step 3 — what does the nginx error log say?
tail -20 /var/log/nginx/error.log
# [error] 1235: open() "/var/log/nginx/access.log" failed (28: No space left)
# Confirmed — nginx cannot write logs because disk is full

# Step 4 — compress old logs to free space
gzip /var/log/nginx/access.log
# Now access.log is access.log.gz — 95% smaller

# Step 5 — tell nginx to reopen its log files
sudo systemctl reload nginx

# Step 6 — verify it is writing again
tail -f /var/log/nginx/access.log
# 192.168.1.10 GET /api/products 200 512   ← writing again

# Step 7 — verify the webstore is serving correctly
curl -I http://localhost
# HTTP/1.1 200 OK

# Step 8 — prevent this from happening again
# Compress logs older than 7 days automatically
find /var/log/nginx/ -name "*.log" -mtime +7 -exec gzip {} \;
```

---

## What breaks

| Symptom | Layer | First command |
|---|---|---|
| Service failing silently | L1/L3 | `journalctl -u <svc> -n 50` |
| Disk full — services writing logs | L3 | `df -h` then `du -sh /var/log/* \| sort -rh` |
| OOM kill — process killed silently | L0/L3 | `dmesg \| grep -i oom` |
| Cannot SSH in | L2/L4 | Check `journalctl -u sshd` from console |
| journalctl shows no logs | L1/L3 | `systemctl status systemd-journald` — journal may have stopped |
| `/var/log` filling with old logs | L3 | `find /var/log -mtime +30 -name "*.log" -exec gzip {} \;` |

---

## Daily commands

| Command | What it does |
|---|---|
| `journalctl -u <svc> -f` | Follow live logs for a service |
| `journalctl -u <svc> -n 50` | Last 50 lines of service logs |
| `journalctl -b -p err` | All errors since last boot |
| `journalctl -b -1` | All logs from previous boot — useful after a crash |
| `df -h` | Check disk space on all filesystems |
| `du -sh /var/log/* \| sort -rh \| head -10` | Find the largest log files |
| `cat /run/nginx.pid` | Read PID of a running service from its PID file |
| `ls -la /run/docker.sock` | Confirm Docker socket exists and is accessible |
| `find /var/log -mtime +30 -name "*.log" -exec gzip {} \;` | Compress logs older than 30 days |
| `grep "$(date +'%b %e')" /var/log/syslog \| grep -i error` | Find today's errors in syslog |

---

→ **Interview questions for this topic:** [99-interview-prep → Logs & Debug](../99-interview-prep/README.md#logs-and-debug)
