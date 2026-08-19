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

# Filter Commands

> **Layer:** L5 — Tools & Files   
> **Depends on:** [03 Working with Files](../03-working-with-files/README.md) — you need to be able to read and navigate files before filtering them   
> **Used in production when:** Something broke and you need to search thousands of log lines to find the one that matters — without opening a text editor   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [The webstore access log](#the-webstore-access-log)
- [1. The Pipe — how everything connects](#1-the-pipe--how-everything-connects)
- [2. grep — Search File Contents](#2-grep--search-file-contents)
- [3. find — Search the Filesystem](#3-find--search-the-filesystem)
- [4. locate — Fast Name Lookup](#4-locate--fast-name-lookup)
- [5. wc — Count Lines, Words, Bytes](#5-wc--count-lines-words-bytes)
- [6. cut — Extract Fields](#6-cut--extract-fields)
- [7. sort — Order Lines](#7-sort--order-lines)
- [8. uniq — Deduplicate Lines](#8-uniq--deduplicate-lines)
- [9. tr — Translate Characters](#9-tr--translate-characters)
- [10. tee — Split a Stream](#10-tee--split-a-stream)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

A production server generates thousands of log lines every hour. You will never open them in a text editor. You will never scroll through them manually. Filter commands let you search, slice, count, sort, and chain operations against any file or stream directly from the terminal. The pipe `|` connects them into analysis chains that answer real questions in seconds. This is how a DevOps engineer reads a system without a GUI.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       grep · find · cut · sort · uniq · wc · tee · tr · pipe
  L4  Config
  L3  State & Debug   ← /var/log — the logs you filter live here
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

The logs at L3 are the raw material. The filter commands at L5 are the tools that make sense of them. Every incident investigation you run in file 14 (Logs & Debug) uses the commands you learn here.

---

## The webstore access log

Every example in this file uses this log. Save it to `~/webstore/logs/access.log` to follow along.

```
192.168.1.10 GET /api/products 200
192.168.1.11 GET /api/products 200
192.168.1.12 POST /api/orders 201
192.168.1.10 GET /api/products 200
192.168.1.13 GET /api/users 404
192.168.1.14 POST /api/orders 500
192.168.1.11 GET /api/products 200
192.168.1.15 DELETE /api/orders/7 403
192.168.1.10 GET /api/products 200
192.168.1.14 POST /api/orders 500
```

Fields: `IP  METHOD  PATH  STATUS`

---

## 1. The Pipe — how everything connects

The pipe `|` takes the output of one command and feeds it directly as input to the next. No temporary files. No intermediate steps. It turns single commands into analysis chains.

```
command1 | command2 | command3
```

Think of it as an assembly line. Each command does one job. The pipe passes the result to the next worker. The final output is the answer to your question.

```bash
# Question: how many 500 errors are in the log?
grep '500' ~/webstore/logs/access.log | wc -l
# 2

# Question: which IPs caused the 500 errors?
grep '500' ~/webstore/logs/access.log | cut -d' ' -f1
# 192.168.1.14
# 192.168.1.14
```

Every section below is a tool you add to your pipeline vocabulary.

---

## 2. grep — Search File Contents

`grep` (Global Regular Expression Print) searches inside files for lines matching a pattern. It is the most-used command in incident investigation. Every log analysis starts here.

| Flag | Full form | What it does | Example |
|---|---|---|---|
| `grep <pat> <file>` | — | Find lines matching pattern — case sensitive | `grep '500' access.log` |
| `-i` | --ignore-case | Case-insensitive match | `grep -i 'error' access.log` |
| `-n` | --line-number | Show line numbers alongside matches | `grep -n '500' access.log` |
| `-c` | --count | Count matching lines, do not print them | `grep -c '500' access.log` |
| `-v` | --invert-match | Show lines that do NOT match | `grep -v '200' access.log` |
| `-w` | --word-regexp | Match whole words only | `grep -w 'GET' access.log` |
| `-r` | --recursive | Search all files in a directory | `grep -r 'db_host' ~/webstore/config/` |

```bash
# Find all 500 errors
grep '500' ~/webstore/logs/access.log
# 192.168.1.14 POST /api/orders 500
# 192.168.1.14 POST /api/orders 500

# Count how many 500 errors occurred
grep -c '500' ~/webstore/logs/access.log
# 2

# Surface every non-200 request — all problems at once
grep -v '200' ~/webstore/logs/access.log
# 192.168.1.12 POST /api/orders 201
# 192.168.1.13 GET /api/users 404
# 192.168.1.14 POST /api/orders 500
# 192.168.1.15 DELETE /api/orders/7 403
# 192.168.1.14 POST /api/orders 500

# Search every log file in the directory
grep -r '500' ~/webstore/logs/
# access.log:192.168.1.14 POST /api/orders 500
# access.log:192.168.1.14 POST /api/orders 500
```

`grep -v '200'` on any access log immediately surfaces every non-successful request. You do not scroll — you filter.

---

## 3. find — Search the Filesystem

`find` walks the directory tree in real time and returns every file matching your criteria. Results are always current — it reads the live filesystem, not a cache.

| Option | What it does | Example |
|---|---|---|
| `-name "*.log"` | Match by filename with wildcards | `find ~/webstore/logs -name "*.log"` |
| `-type f` | Regular files only | `find ~/webstore -type f` |
| `-type d` | Directories only | `find ~/webstore -type d` |
| `-mtime +7` | Modified more than 7 days ago | `find ~/webstore/logs -mtime +7` |
| `-mtime -1` | Modified in the last 24 hours | `find ~/webstore/logs -mtime -1` |
| `-size +1M` | Larger than 1 megabyte | `find ~/webstore/logs -size +1M` |
| `-exec <cmd> {} \;` | Run a command on every match | `find ~/webstore -name "*.tmp" -exec rm {} \;` |

```bash
# Find the webstore config wherever it is
find ~/webstore -name "webstore.conf"
# /home/akhil/webstore/config/webstore.conf

# Find log files modified in the last day
find ~/webstore/logs -mtime -1 -name "*.log"
# /home/akhil/webstore/logs/access.log

# Find and delete all temp files left by a crashed process
find ~/webstore -name "*.tmp" -exec rm {} \;

# Find large log files consuming disk space
find ~/webstore/logs -size +100M
```

---

## 4. locate — Fast Name Lookup

`locate` searches a prebuilt database of filenames. Results are instant but only as fresh as the last time `updatedb` ran — usually once a day.

| Option | Full form | What it does |
|---|---|---|
| `locate <name>` | — | Find all paths containing this name |
| `-i` | --ignore-case | Case-insensitive match |
| `-l 5` | --limit | Limit results to 5 |
| `-c` | --count | Count matches only |

```bash
locate webstore.conf
# /home/akhil/webstore/config/webstore.conf

# File created in the last hour and locate cannot find it?
sudo updatedb && locate webstore.conf
```

**find vs locate — when to use which:**

| | find | locate |
|---|---|---|
| Results | Always current | Only as fresh as last `updatedb` |
| Speed | Slower on large trees | Instant |
| Filters | Name, type, size, age, owner | Name only |
| Use when | You need exact current results | You just need to know if a file exists |

---

## 5. wc — Count Lines, Words, Bytes

`wc` (Word Count) counts lines, words, and bytes in a file or stream.

| Flag | Full form | What it counts |
|---|---|---|
| `wc -l` | --lines | Lines only — most useful |
| `wc -w` | --words | Words only |
| `wc -c` | --bytes | Bytes only |

```bash
# How many lines in the access log?
wc -l ~/webstore/logs/access.log
# 10 /home/akhil/webstore/logs/access.log

# How many 500 errors? (in a pipeline)
grep '500' ~/webstore/logs/access.log | wc -l
# 2
```

`wc -l` at the end of any pipeline tells you how many results the previous command produced.

---

## 6. cut — Extract Fields

`cut` extracts specific columns from structured text. You define the delimiter with `-d` and which field to keep with `-f`. Fields are numbered from 1.

| Option | Full form | What it does |
|---|---|---|
| `-d' ' -f1` | --delimiter --fields | Split on space, take field 1 |
| `-d',' -f2` | --delimiter --fields | Split on comma, take field 2 |
| `-d' ' -f1,4` | --delimiter --fields | Take fields 1 and 4 |

```bash
# Extract IP addresses (field 1 — space delimited)
cut -d' ' -f1 ~/webstore/logs/access.log
# 192.168.1.10
# 192.168.1.11
# 192.168.1.12
# ...

# Extract status codes (field 4)
cut -d' ' -f4 ~/webstore/logs/access.log
# 200
# 200
# 201
# ...

# Extract IP and status code together
cut -d' ' -f1,4 ~/webstore/logs/access.log
# 192.168.1.10 200
# 192.168.1.11 200
# ...
```

---

## 7. sort — Order Lines

`sort` orders lines of text. It almost always appears before `uniq` — `uniq` only removes adjacent duplicates, so you must sort first to bring identical lines together.

| Flag | Full form | What it does |
|---|---|---|
| `sort` | — | Alphabetical ascending |
| `-r` | --reverse | Reverse order |
| `-n` | --numeric-sort | Sort numerically, not alphabetically |
| `-rn` | --reverse --numeric-sort | Largest numbers first |
| `-k <N>` | --key | Sort by field N |

```bash
# Sort the log by status code (field 4)
sort -k4 ~/webstore/logs/access.log
# 192.168.1.12 POST /api/orders 201
# 192.168.1.15 DELETE /api/orders/7 403
# 192.168.1.13 GET /api/users 404
# 192.168.1.14 POST /api/orders 500
# 192.168.1.14 POST /api/orders 500
# 192.168.1.10 GET /api/products 200
# ...
```

---

## 8. uniq — Deduplicate Lines

`uniq` removes or counts duplicate **consecutive** lines. Always run `sort` first.

| Flag | Full form | What it does |
|---|---|---|
| `uniq` | — | Remove consecutive duplicate lines |
| `-c` | --count | Prefix each line with its occurrence count |
| `-d` | --repeated | Show only lines that appeared more than once |
| `-u` | --unique | Show only lines that appeared exactly once |

**The classic combination — ranked hit count per IP:**

```bash
cut -d' ' -f1 ~/webstore/logs/access.log | sort | uniq -c | sort -rn
#   5 192.168.1.10
#   2 192.168.1.11
#   2 192.168.1.14
#   1 192.168.1.12
#   1 192.168.1.13
#   1 192.168.1.15
```

Read left to right: extract IPs → sort so identical IPs are adjacent → count and deduplicate → sort by count descending. Result: ranked list of who is hitting the API most. This same pattern works on any field — endpoints, status codes, methods.

---

## 9. tr — Translate Characters

`tr` (Translate) replaces or deletes characters in a stream. It reads from stdin — feed it with a pipe.

| Option | Full form | What it does |
|---|---|---|
| `tr 'a-z' 'A-Z'` | — | Uppercase all lowercase letters |
| `-d '0-9'` | --delete | Delete all digits |
| `-s ' '` | --squeeze-repeats | Collapse multiple spaces into one |

```bash
# Uppercase the entire log for case-insensitive comparison
cat ~/webstore/logs/access.log | tr 'a-z' 'A-Z'

# Remove digits from a stream
echo "error404" | tr -d '0-9'
# error
```

Most useful in pipelines when you need to normalise text before passing it to another command.

---

## 10. tee — Split a Stream

`tee` reads from stdin and writes to both stdout and a file simultaneously. You see the output on screen and it gets saved — without running the command twice.

| Flag | Full form | What it does |
|---|---|---|
| `tee <file>` | — | Write to stdout and file, overwriting file |
| `tee -a <file>` | --append | Write to stdout and append to file |

```bash
# Save all 500 errors to a file AND see them on screen
grep '500' ~/webstore/logs/access.log | tee ~/webstore/logs/errors.log
# 192.168.1.14 POST /api/orders 500   ← printed to terminal
# 192.168.1.14 POST /api/orders 500   ← also written to errors.log
```

Use `tee` when you want a record of your investigation without losing the ability to keep piping.

---

## On the webstore

These are the pipelines you actually run during a webstore incident.
Each one answers a specific question you will be asked.

```bash
# Question 1 — how many errors hit the API in this log?
grep '500' ~/webstore/logs/access.log | wc -l
# 2

# Question 2 — which IP is generating all the 500 errors?
grep '500' ~/webstore/logs/access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn
#   2 192.168.1.14

# Question 3 — which endpoints are getting hit most?
cut -d' ' -f3 ~/webstore/logs/access.log | sort | uniq -c | sort -rn
#   5 /api/products
#   2 /api/orders
#   1 /api/users
#   1 /api/orders/7

# Question 4 — show every non-200 request with line numbers
grep -vn '200' ~/webstore/logs/access.log
# 3:192.168.1.12 POST /api/orders 201
# 5:192.168.1.13 GET /api/users 404
# 6:192.168.1.14 POST /api/orders 500
# 8:192.168.1.15 DELETE /api/orders/7 403
# 10:192.168.1.14 POST /api/orders 500

# Question 5 — save all errors to a separate file for the team
grep -v '200' ~/webstore/logs/access.log | tee ~/webstore/logs/non-200.log
# (prints to screen and saves to file simultaneously)

# Question 6 — find all log files changed in the last 24 hours
find ~/webstore/logs -mtime -1 -name "*.log"
# /home/akhil/webstore/logs/access.log

# Question 7 — find which log files contain 500 errors
find ~/webstore/logs -name "*.log" -exec grep -l '500' {} \;
# /home/akhil/webstore/logs/access.log
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `grep` returns nothing when you expected matches | Pattern is case-sensitive and case does not match | Add `-i` for case-insensitive matching |
| `grep '500'` also matches `5000` or `15001` | Pattern matches anywhere in the line | Use `-w` for whole-word match or anchor with `\b500\b` |
| `uniq -c` not deduplicating correctly | Identical lines are not adjacent | Run `sort` before `uniq` — always |
| `cut` returns the wrong field | Fields are numbered from 1, not 0, and the delimiter may contain multiple spaces | Check the actual delimiter with `cat -A file` to see whitespace |
| `find -exec` errors with `missing argument to -exec` | Missing `\;` at the end of the exec block | Always close `-exec <cmd> {} \;` with `\;` |
| `locate` cannot find a file you just created | Database is stale — locate uses a cache | Run `sudo updatedb` then try again |
| Pipeline produces no output | An early command in the chain matched nothing | Test each command individually before piping |

---

## Daily commands

| Command | What it does |
|---|---|
| `grep '<pat>' <file>` | Find lines matching a pattern |
| `grep -v '<pat>' <file>` | Find lines that do NOT match — surfaces all problems |
| `grep -rn '<pat>' <dir>` | Search all files in a directory, show line numbers |
| `find <dir> -name "<pat>"` | Find files by name in real time |
| `find <dir> -mtime -1` | Find files modified in the last 24 hours |
| `cut -d' ' -f<N> <file>` | Extract field N from space-delimited text |
| `sort \| uniq -c \| sort -rn` | Count and rank occurrences — the core analysis pattern |
| `wc -l` | Count lines — always useful at the end of a pipeline |
| `tee <file>` | Save pipeline output to file while still seeing it on screen |
| `cmd1 \| cmd2 \| cmd3` | Chain commands — each feeds the next |

---

→ **Interview questions for this topic:** [99-interview-prep → Filter Commands](../99-interview-prep/README.md#filter-commands)
