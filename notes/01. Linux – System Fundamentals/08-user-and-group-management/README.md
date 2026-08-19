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

# User & Group Management

> **Layer:** L4 — Config   
> **Depends on:** [02 Basics](../02-basics/README.md) — you need `whoami` and `id` before managing other users   
> **Used in production when:** Setting up a new server, adding a team member, creating a service account for nginx or docker, or auditing who has access to what   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. How Linux identifies users](#1-how-linux-identifies-users)
- [2. Key system files](#2-key-system-files)
- [3. UID ranges — who is who](#3-uid-ranges--who-is-who)
- [4. User management](#4-user-management)
- [5. Group management](#5-group-management)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

Every process on a Linux server runs as a user. Every file is owned by a user and a group. This is not bureaucracy — it is the access control model that prevents a compromised web server from reading your database credentials, and prevents a developer's script from accidentally deleting system files. When nginx serves the webstore frontend, it runs as `www-data` — a system user with no shell, no home directory, and read-only access to exactly the files it needs. Understanding users and groups is understanding who is allowed to do what on the machine.

---

## How it fits the stack

```
  L6  You  ← /home/akhil  /home/charan  /home/pramod  /home/navya  /home/indhu
  L5  Tools & Files
  L4  Config  ← this file lives here
       /etc/passwd  /etc/shadow  /etc/group  /etc/gshadow
  L3  State & Debug
  L2  Networking
  L1  Process Manager  ← services run as specific users defined at L4
  L0  Kernel & Hardware
```

Users are defined at L4 (/etc). They live at L6 (/home). Services at L1 run under those users. Permissions in file 09 enforce what each user can access. All four layers are connected by this one concept.

---

## 1. How Linux identifies users

Linux does not track users by name — it tracks them by **UID** (User ID), a number. When you run `ls -l` and see `akhil` as the owner, Linux is storing UID `1000` and your terminal is resolving it to a name for readability. The same is true for groups — every group has a **GID** (Group ID).

Every process has a UID. That UID determines what files the process can read, write, or execute. A process running as root (UID 0) can access anything. This is why services should never run as root — a compromised root process means full system compromise.

---

## 2. Key system files

These four files define every user and group on the system. Read them often. Never edit them directly — use the commands in section 4 and 5 instead.

| File | What it contains | Who can read |
|---|---|---|
| `/etc/passwd` | One line per user: username, UID, GID, home dir, shell | Everyone |
| `/etc/shadow` | Hashed passwords and password aging | Root only |
| `/etc/group` | One line per group: name, GID, member list | Everyone |
| `/etc/gshadow` | Encrypted group passwords and admins | Root only |

**Reading `/etc/passwd`:**

```bash
grep www-data /etc/passwd
# www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
#          │     │  │        │         └── shell: nologin = cannot log in interactively
#          │     │  │        └── home directory
#          │     │  └── GID
#          │     └── UID
#          └── x = password is in /etc/shadow
```

`/usr/sbin/nologin` as the shell means this user cannot log in interactively. Intentional for service accounts — nginx does not need a shell.

**Reading `/etc/group`:**

```bash
grep webstore-team /etc/group
# webstore-team:x:3000:www-data,akhil,charan,pramod
#               │  │    └── comma-separated member list
#               │  └── GID
#               └── password placeholder
```

---

## 3. UID ranges — who is who

| Range | Purpose | Examples |
|---|---|---|
| `0` | Root — full system access | `root` |
| `1–999` | System accounts — services, no login shell | `www-data` (33), `postgres` (999) |
| `1000+` | Human users — login shell, home directory | `akhil` (1000), `charan` (1001) |

When you install nginx, it creates `www-data` automatically in the system range. When you create `akhil`, they get UID 1000+. This separation is intentional — services and humans should never share an identity.

---

## 4. User management

**Creating a user:**

```bash
# -m (--create-home) creates /home/akhil
# -s (--shell) sets the login shell
sudo useradd -m -s /bin/bash akhil

# Set the password
sudo passwd akhil
# New password:
# Retype new password:
# passwd: password updated successfully
```

**Modifying a user:**

```bash
# -aG (--append --groups) adds to a group — the -a is critical
sudo usermod -aG webstore-team akhil

# Without -a it REPLACES all groups — user loses all other access
# Always use -aG never -G alone

# Change login shell
sudo usermod -s /bin/bash akhil

# Change username
sudo usermod -l charan-new charan
```

**Deleting a user:**

```bash
# Delete user, keep /home/akhil — useful when preserving files
sudo userdel akhil

# Delete user AND remove /home/akhil and mail spool
sudo userdel --remove akhil
```

**Checking users:**

```bash
# Your UID, GID, and every group you belong to
id
# uid=1000(akhil) gid=1000(akhil) groups=1000(akhil),27(sudo),3000(webstore-team)

# Another user's info
id charan

# All groups a user belongs to
groups akhil
# akhil : akhil sudo webstore-team docker
```

---

## 5. Group management

Groups give multiple users the same access to a resource without duplicating permissions. Instead of giving akhil, charan, and pramod individual write access to the webstore directory, you create `webstore-team`, give the directory group write access, and add all three to the group.

```bash
# Create a group
sudo groupadd webstore-team

# Create with specific GID
sudo groupadd -g 3000 webstore-team

# Add a user to a group
sudo gpasswd -a akhil webstore-team
sudo gpasswd -a charan webstore-team
sudo gpasswd -a pramod webstore-team

# Remove a user from a group
sudo gpasswd -d akhil webstore-team

# Rename a group
sudo groupmod -n webstore-devs webstore-team

# Delete a group
sudo groupdel webstore-devs

# Confirm group membership
getent group webstore-team
# webstore-team:x:3000:akhil,charan,pramod

# Apply new group in current session without re-login
newgrp webstore-team
```

---

## On the webstore

The webstore needs specific users and groups set up before permissions can be locked down in file 09.

```bash
# Step 1 — confirm nginx is running as www-data
ps aux | grep nginx
# www-data  1234  ...  nginx: worker process

# Step 2 — create the webstore team group
sudo groupadd -g 3000 webstore-team

# Step 3 — add the developers
sudo gpasswd -a akhil webstore-team
sudo gpasswd -a charan webstore-team
sudo gpasswd -a pramod webstore-team

# Step 4 — add www-data so nginx can read webstore files
sudo gpasswd -a www-data webstore-team

# Step 5 — navya and indhu are designers, different group
sudo groupadd -g 3001 design-team
sudo gpasswd -a navya design-team
sudo gpasswd -a indhu design-team

# Step 6 — only akhil gets sudo
# (akhil was given sudo during initial user creation — verify)
groups akhil
# akhil : akhil sudo webstore-team

# Step 7 — confirm the full webstore-team membership
getent group webstore-team
# webstore-team:x:3000:akhil,charan,pramod,www-data

# Step 8 — verify nobody else has unexpected access
cat /etc/passwd | grep -v nologin | grep -v false | awk -F: '$3 >= 1000 {print $1, $3}'
# akhil 1000
# charan 1001
# pramod 1002
# navya 1003
# indhu 1004
```

File 09 (Permissions) picks up from here — now that the right users and groups exist, you can lock down which directories each one can access.

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `useradd: user already exists` | Username taken | Pick a different name or `userdel` the old one first |
| `usermod -G group user` removed all other groups | Used `-G` without `-a` | Always use `-aG` — `-G` alone replaces all group memberships |
| Group change not taking effect in current session | Linux caches group membership at login | Run `newgrp <group>` or log out and back in |
| `userdel: user is currently logged in` | User has an active session | `pkill -u username` to kill their session first |
| Service still runs as wrong user after config change | systemd caches the unit — restart needed | `sudo systemctl daemon-reload && sudo systemctl restart <service>` |
| `getent group <group>` shows no members | Users were added with `usermod -aG` but didn't re-login | Check with `id <user>` — shows current session groups. Have them re-login. |

---

## Daily commands

| Command | What it does |
|---|---|
| `sudo useradd -m -s /bin/bash <user>` | Create user with home directory and bash shell |
| `sudo passwd <user>` | Set or change a user's password |
| `sudo usermod -aG <group> <user>` | Add user to group — always use `-aG` never `-G` alone |
| `sudo userdel --remove <user>` | Delete user and their home directory |
| `sudo groupadd <group>` | Create a new group |
| `sudo gpasswd -a <user> <group>` | Add user to group |
| `sudo gpasswd -d <user> <group>` | Remove user from group |
| `getent group <group>` | Show group members |
| `groups <user>` | List all groups a user belongs to |
| `newgrp <group>` | Switch active group in current session without re-login |

---

→ **Interview questions for this topic:** [99-interview-prep → Users & Groups](../99-interview-prep/README.md#users-and-groups)
