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

# sed — Stream Editor

> **Layer:** L5 — Tools & Files   
> **Depends on:** [04 Filter Commands](../04-filter-commands/README.md) — you need grep and pipes before sed makes sense   
> **Used in production when:** A deploy script needs to update a config file without opening an editor — swap `env=production` to `env=staging`, update a port, strip comment lines before parsing   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [The webstore config file](#the-webstore-config-file)
- [1. How sed works](#1-how-sed-works)
- [2. Substitution — the core operation](#2-substitution--the-core-operation)
- [3. Targeting specific lines](#3-targeting-specific-lines)
- [4. In-place editing](#4-in-place-editing)
- [5. Deleting lines](#5-deleting-lines)
- [6. Printing specific lines](#6-printing-specific-lines)
- [7. Inserting and appending lines](#7-inserting-and-appending-lines)
- [8. Running multiple commands](#8-running-multiple-commands)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

`grep` finds lines. `cut` extracts fields. `sed` transforms content. It reads a file or stream one line at a time, applies your editing instruction, and outputs the result — without opening an editor, without moving a cursor, without touching the file unless you tell it to. You describe the change once and sed applies it to every matching line. This is how deploy scripts update config files, how you strip comment lines before parsing, and how you make the same change across hundreds of lines in one command.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       sed — transforms files and streams without opening an editor
  L4  Config  ← /etc/webstore/webstore.conf — the files sed edits
  L3  State & Debug
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

sed sits between you and the config files at L4. When a deploy script needs to switch an environment value or update a port in `/etc/nginx/nginx.conf`, sed is the tool that makes that change programmatically — no manual editing, no risk of human error, reproducible every time.

---

## The webstore config file

Every example in this file uses this config. It lives at `~/webstore/config/webstore.conf` — you created it in file 03.

```
db_host=webstore-db
db_port=5432
api_port=8080
api_host=webstore-api
frontend_port=80
frontend_host=webstore-frontend
env=production
```

---

## 1. How sed works

sed reads a file one line at a time. For each line it checks whether your pattern matches, applies the instruction if it does, then prints the result. By default it prints every line — changed or not. The original file is untouched unless you use `-i`.

```
sed 'instruction' file

instruction = [address] command
  address = which lines to act on (optional — default is all lines)
  command = what to do (substitute, delete, print, insert)
```

**Key flags:**

| Flag | Full form | What it does |
|---|---|---|
| `-n` | --quiet | Suppress automatic printing — only print lines you explicitly request with `p` |
| `-i` | --in-place | Write changes back to the original file |
| `-e` | --expression | Chain multiple instructions in one command |

---

## 2. Substitution — the core operation

The substitution command is the one you use 90% of the time:

```
s/OLD/NEW/flags
│ │   │   │
│ │   │   └── flags: g = global (all matches), p = print changed lines
│ │   └────── replacement text
│ └────────── pattern to find
└──────────── s = substitute
```

**Replace first match per line:**

```bash
sed 's/production/staging/' ~/webstore/config/webstore.conf
# db_host=webstore-db
# db_port=5432
# api_port=8080
# api_host=webstore-api
# frontend_port=80
# frontend_host=webstore-frontend
# env=staging          ← only this line changed
```

The file is not modified — output goes to terminal only.

**Replace all matches per line with `g` (global):**

```bash
sed 's/webstore/mystore/g' ~/webstore/config/webstore.conf
# db_host=mystore-db
# api_host=mystore-api
# frontend_host=mystore-frontend
# env=production
```

Without `g`, only the first match per line is replaced. With `g`, every match on every line is replaced.

**When the pattern contains `/`, use a different delimiter:**

```bash
# This breaks — forward slash conflicts with the sed delimiter
sed 's/api/v2/api/g' webstore.conf

# Use # as delimiter instead — any char not in your pattern works
sed 's#/api#/v2/api#g' webstore.conf

# Other common choices: | and @
sed 's|production|staging|g' webstore.conf
```

---

## 3. Targeting specific lines

By default sed acts on every line. You can restrict it with a line number, a range, or a pattern.

**Line number:**

```bash
# Replace only on line 1
sed '1 s/db_host/database_host/' ~/webstore/config/webstore.conf
# database_host=webstore-db   ← only line 1 changed
# db_port=5432
# ...
```

**Line range:**

```bash
# Replace on lines 1 through 3 only
sed '1,3 s/webstore/mystore/' ~/webstore/config/webstore.conf
# db_host=mystore-db
# db_port=5432
# api_port=8080
# api_host=webstore-api       ← unchanged, line 4
```

**From line N to end of file (`$` = last line):**

```bash
sed '3,$ s/webstore/mystore/' ~/webstore/config/webstore.conf
```

**Only lines matching a pattern:**

```bash
# Only replace on lines that contain "port"
sed '/port/ s/8080/9090/' ~/webstore/config/webstore.conf
# db_host=webstore-db
# db_port=5432               ← contains "port" but 5432 not matched
# api_port=9090              ← changed
# frontend_port=80           ← contains "port" but 80 not matched
```

**Print only lines where substitution happened (`-n` + `p` flag):**

```bash
sed -n 's/production/staging/p' ~/webstore/config/webstore.conf
# env=staging
```

`-n` suppresses all output. `p` prints only lines that were actually changed. Together: confirmation of exactly what sed touched.

---

## 4. In-place editing

Everything above only prints the result — the original file is not modified. `-i` writes changes back to the file.

```bash
# Change production to staging directly in the file
sed -i 's/production/staging/' ~/webstore/config/webstore.conf
```

After this, `webstore.conf` is permanently changed. No undo unless you have a backup.

**Always back up before in-place editing:**

```bash
# Step 1 — back up first
cp ~/webstore/config/webstore.conf ~/webstore/backup/webstore.conf.bak

# Step 2 — then edit in-place
sed -i 's/production/staging/' ~/webstore/config/webstore.conf

# Step 3 — verify the change landed
grep 'env' ~/webstore/config/webstore.conf
# env=staging
```

> **macOS difference:** `-i` requires an empty string argument on macOS: `sed -i '' 's/old/new/' file`. On Linux no empty string is needed. If your script runs on both, use `sed -i'' 's/old/new/' file` — works on both.

---

## 5. Deleting lines

```bash
# Delete all lines containing "frontend"
sed '/frontend/d' ~/webstore/config/webstore.conf
# db_host=webstore-db
# db_port=5432
# api_port=8080
# api_host=webstore-api
# env=production

# Delete lines starting with # (strip all comments)
sed '/^#/d' ~/webstore/config/webstore.conf

# Delete the last line ($ = last line, d = delete)
sed '$d' ~/webstore/config/webstore.conf

# Delete lines 5 through end
sed '5,$d' ~/webstore/config/webstore.conf
```

`^` matches the start of a line. `$` alone means the last line. `/pattern/d` deletes every line containing that pattern.

---

## 6. Printing specific lines

With `-n` and `p` you can extract exactly the lines you need — more targeted than `head` or `tail`.

```bash
# Print only lines 2 through 4
sed -n '2,4p' ~/webstore/config/webstore.conf
# db_port=5432
# api_port=8080
# api_host=webstore-api

# Print only lines containing "api"
sed -n '/api/p' ~/webstore/config/webstore.conf
# api_port=8080
# api_host=webstore-api
```

---

## 7. Inserting and appending lines

```bash
# Insert a line BEFORE line 1 (i = insert)
sed '1i\# webstore config — do not edit manually' ~/webstore/config/webstore.conf
# # webstore config — do not edit manually   ← inserted before line 1
# db_host=webstore-db
# ...

# Append a line AFTER the last line (a = append)
sed '$a\log_level=info' ~/webstore/config/webstore.conf
# ...
# env=production
# log_level=info   ← appended after last line

# Insert in-place — write it back
sed -i '1i\# webstore config — do not edit manually' ~/webstore/config/webstore.conf
```

---

## 8. Running multiple commands

`-e` chains multiple instructions in a single sed pass. One read of the file, multiple transformations applied.

```bash
# Swap environment AND update the API port in one command
sed -e 's/production/staging/' -e 's/api_port=8080/api_port=9090/' ~/webstore/config/webstore.conf
# db_host=webstore-db
# db_port=5432
# api_port=9090              ← changed
# api_host=webstore-api
# frontend_port=80
# frontend_host=webstore-frontend
# env=staging                ← changed
```

Cleaner than running sed twice and faster on large files — the file is read only once.

---

## On the webstore

Deploy day. The webstore needs to switch from production to staging config.
You update the config file programmatically — no editor, no manual steps.

```bash
# Step 1 — confirm current state
cat ~/webstore/config/webstore.conf
# env=production
# api_port=8080

# Step 2 — back up before any change
cp ~/webstore/config/webstore.conf ~/webstore/backup/webstore.conf.bak

# Step 3 — switch to staging in one command
sed -i -e 's/production/staging/' -e 's/api_port=8080/api_port=9090/' \
  ~/webstore/config/webstore.conf

# Step 4 — verify both changes landed
grep -E 'env|api_port' ~/webstore/config/webstore.conf
# api_port=9090
# env=staging

# Step 5 — add a header comment to the config
sed -i '1i\# webstore staging config — generated by deploy script' \
  ~/webstore/config/webstore.conf

# Step 6 — verify the final file
cat ~/webstore/config/webstore.conf
# # webstore staging config — generated by deploy script
# db_host=webstore-db
# db_port=5432
# api_port=9090
# api_host=webstore-api
# frontend_port=80
# frontend_host=webstore-frontend
# env=staging

# Step 7 — to roll back, restore from backup
cp ~/webstore/backup/webstore.conf.bak ~/webstore/config/webstore.conf
grep 'env' ~/webstore/config/webstore.conf
# env=production
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `sed: 1: "s/old/new/"`: extra characters | macOS sed requires `sed -i ''` not `sed -i` | Use `sed -i ''` on macOS, `sed -i` on Linux |
| Substitution silently does nothing | Pattern does not match — case mismatch or wrong field | Run without `-i` first to preview output, add `-i` only when confirmed |
| `s/path/to/file/replacement/` breaks | Forward slash in pattern conflicts with delimiter | Switch delimiter: `s#path/to/file#replacement#` |
| `-i` changed the file and you cannot undo | No backup was made | Restore from version control or backup — this is why you `cp` first |
| `uniq` not working after sed | Lines not adjacent after transformation | Add `sort` before `uniq` in the pipeline |
| `sed -n '/pattern/p'` prints nothing | Pattern does not match | Test the pattern with `grep 'pattern' file` first to confirm it works |
| Multi-line sed produces unexpected output | sed processes one line at a time by default | For multi-line transformations use `N` command or switch to awk |

---

## Daily commands

| Command | What it does |
|---|---|
| `sed 's/OLD/NEW/' <file>` | Replace first match per line — preview only, no file change |
| `sed 's/OLD/NEW/g' <file>` | Replace all matches per line — preview only |
| `sed -n 's/OLD/NEW/p' <file>` | Preview only the lines that would change |
| `sed -i 's/OLD/NEW/' <file>` | Replace and write back to file — always back up first |
| `sed -i -e 's/A/B/' -e 's/C/D/' <file>` | Multiple replacements in one pass |
| `sed '/^#/d' <file>` | Strip all comment lines |
| `sed -n '2,4p' <file>` | Print only lines 2 through 4 |
| `sed -n '/pattern/p' <file>` | Print only lines matching pattern |
| `sed '$a\new line' <file>` | Append a line at the end |
| `sed 's#/old/path#/new/path#g' <file>` | Replace paths — use `#` delimiter to avoid slash conflicts |

---

→ **Interview questions for this topic:** [99-interview-prep → sed](../99-interview-prep/README.md#sed)
