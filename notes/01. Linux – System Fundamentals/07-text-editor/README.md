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

# vim — Terminal Text Editor

> **Layer:** L5 — Tools & Files   
> **Depends on:** [02 Basics](../02-basics/README.md) — you need to navigate the filesystem before editing files on it   
> **Used in production when:** You SSH into a server and need to edit a config file — no GUI, no VS Code, just the terminal   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. The three modes](#1-the-three-modes)
- [2. Opening and exiting](#2-opening-and-exiting)
- [3. Navigation](#3-navigation)
- [4. Editing](#4-editing)
- [5. Search and replace](#5-search-and-replace)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

On a remote Linux server there is no GUI. No VS Code, no Sublime, no Notepad. When you need to edit a config file, fix a broken nginx config, or write a quick script, you use a terminal editor. `vim` is the one you find on every Linux server — it ships with the OS or is one package install away. The reason vim feels hard at first is that it is **modal** — it has separate modes for navigating and for typing. Once that mental model clicks, vim becomes fast. This file gives you everything you need to be productive with vim on a real server.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       vim — the editor you use when no GUI exists
  L4  Config  ← /etc/nginx/ /etc/systemd/ — the files you edit with vim
  L3  State & Debug
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

Every config file at L4 gets edited with vim. The nginx config from file 12, the systemd unit file, the webstore.conf — all opened and changed with the commands in this file.

---

## 1. The three modes

vim starts in **Normal mode** every time you open it. This is the source of most beginner frustration — you open a file, start typing, and nothing appears where you expect it to.

```
Normal mode  ←─────────────── Esc ───────────────┐
     │                                            │
     │  i / a / o                                 │
     ▼                                            │
Insert mode  ── type your content ────────────────┘

Normal mode
     │
     │  :
     ▼
Command-line mode  ── :w  :q  :wq  :q!  :%s/old/new/g
```

| Mode | How to enter | What you do here |
|---|---|---|
| Normal | Default on open, or `Esc` from anywhere | Navigate, delete, copy, paste — keys are commands not text |
| Insert | `i`, `a`, or `o` from Normal | Type text — keyboard behaves like a normal editor |
| Command-line | `:` from Normal | Save, quit, search and replace |

**The rule that prevents most frustration:** when vim is not behaving as expected, press `Esc` first. `Esc` always returns you to Normal mode from anywhere.

---

## 2. Opening and exiting

```bash
# Open a file
vim ~/webstore/config/webstore.conf

# Open and jump directly to line 12 (e.g. error on line 12)
vim +12 ~/webstore/config/nginx.conf

# Open a new file — created on first save
vim ~/webstore/config/new-setting.conf
```

**Exiting — the commands everyone needs first:**

| Command | What it does |
|---|---|
| `:w` | Write (save) the file — stay in vim |
| `:q` | Quit — only works if no unsaved changes |
| `:wq` | Write and quit — save then exit |
| `:q!` | Quit without saving — discard all changes, no questions asked |
| `:x` | Write and quit — same as `:wq` but skips write if nothing changed |

`:q!` is the one you reach for when you opened the wrong file or made changes you want to throw away.

---

## 3. Navigation

In Normal mode the keyboard is for movement. These are the keys you use to move around a file without a mouse.

**Basic movement:**

| Key | Movement |
|---|---|
| `h` | Left one character |
| `l` | Right one character |
| `j` | Down one line |
| `k` | Up one line |
| `w` | Forward one word |
| `b` | Backward one word |
| `0` | Beginning of current line |
| `$` | End of current line |
| `gg` | First line of the file |
| `G` | Last line of the file |
| `NG` | Jump to line N — e.g. `12G` jumps to line 12 |

**Prepend a number to repeat any movement:**
`5j` moves down 5 lines. `3w` jumps forward 3 words. `12G` jumps to line 12.

**When you reach for `NG`:**
An error says "syntax error on line 47 of nginx.conf" — type `47G` in Normal mode and you land exactly there.

---

## 4. Editing

**Entering Insert mode:**

| Key | Where typing begins |
|---|---|
| `i` | Before the cursor |
| `a` | After the cursor |
| `o` | New line below the current line |
| `O` | New line above the current line |

Press `Esc` after typing to return to Normal mode.

**Editing without Insert mode:**

| Command | What it does |
|---|---|
| `x` | Delete the character under the cursor |
| `dd` | Delete (cut) the entire current line |
| `D` | Delete from cursor to end of line |
| `cw` | Delete the current word and enter Insert mode |
| `yy` | Yank (copy) the current line |
| `Nyy` | Yank N lines — `3yy` copies 3 lines |
| `p` | Paste after the cursor / below the current line |
| `u` | Undo the last change |
| `Ctrl+R` | Redo — reverse an undo |

**The most useful editing sequence:** `dd` to cut a line, navigate to where you want it, `p` to paste. Reorder config lines without retyping them.

---

## 5. Search and replace

**Search:**

```
/pattern     search forward — n = next match, N = previous
?pattern     search backward
```

```bash
# Inside vim — search for the api_port line
/api_port
# n to jump to next match, N to go back
```

**Replace — command-line substitute:**

```
:%s/OLD/NEW/g
│  │   │   │
│  │   │   └── g = global, replace all on each line
│  │   └────── replacement
│  └────────── pattern to find
└────────────── % = entire file (without % = current line only)
```

```bash
# Replace every "production" with "staging" in the entire file
:%s/production/staging/g

# Replace with confirmation for each match — vim shows y/n prompt
:%s/production/staging/gc

# Replace only on the current line
:s/8080/9090/g

# Replace only on lines 2 through 5
:2,5s/8080/9090/g
```

---

## On the webstore

```bash
# Scenario 1 — edit the webstore config to change the API port
vim ~/webstore/config/webstore.conf
# /api_port          search for the line
# cw                 delete the word and enter Insert
# api_port=9090      type the new value
# Esc                back to Normal
# :wq                save and quit

# Verify the change
grep 'api_port' ~/webstore/config/webstore.conf
# api_port=9090

# Scenario 2 — nginx config has a syntax error on line 12
vim ~/webstore/config/nginx.conf
# 12G                jump directly to line 12
# fix the error, Esc, :wq

# Scenario 3 — add a new config entry at the end
vim ~/webstore/config/webstore.conf
# G                  jump to last line
# o                  new line below, enter Insert
# log_level=info     type the new entry
# Esc
# :wq

# Scenario 4 — replace the old database hostname everywhere
vim ~/webstore/config/webstore.conf
# :%s/webstore-db-old/webstore-db/g
# :wq

# Verify
grep 'db_host' ~/webstore/config/webstore.conf
# db_host=webstore-db
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| Typed text appears as commands | You are in Normal mode, not Insert | Press `i` to enter Insert mode before typing |
| Can't quit vim | Unsaved changes blocking `:q` | `:wq` to save and quit, or `:q!` to quit without saving |
| `:wq` says "readonly file" | File is owned by root or has no write permission | Quit with `:q!`, then `sudo vim <file>` or fix permissions first |
| Typed `:wq` but text appeared in file | You were in Insert mode when you typed `:` | Press `Esc` first, then `:wq` |
| Search not finding text | Pattern is case-sensitive | Add `\c` for case-insensitive: `/\cpattern` |
| `:%s` replaced wrong text | Pattern matched more than you intended | Use `:%s/old/new/gc` to confirm each replacement before applying |
| `u` undo not working as expected | vim undo history has a limit | Use version control — commit before making large changes |

---

## Daily commands

| Command | What it does |
|---|---|
| `vim <file>` | Open a file in vim |
| `vim +<N> <file>` | Open file and jump to line N |
| `Esc` | Return to Normal mode from anywhere |
| `i` | Enter Insert mode before cursor |
| `o` | New line below, enter Insert mode |
| `:wq` | Save and quit |
| `:q!` | Quit without saving |
| `NG` | Jump to line N in Normal mode |
| `/pattern` | Search forward — `n` next, `N` previous |
| `:%s/OLD/NEW/gc` | Replace all with confirmation |

---

→ **Interview questions for this topic:** [99-interview-prep → vim](../99-interview-prep/README.md#vim)
