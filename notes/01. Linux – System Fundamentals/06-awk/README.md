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

# awk — Text Processing

> **Layer:** L5 — Tools & Files   
> **Depends on:** [05 sed](../05-sed-stream-editor/README.md) — you need pipes, grep, and field thinking before awk.  
> **Used in production when:** You need to calculate totals from a log, build a report from raw text, or filter rows by the exact value of a specific field — things grep and cut cannot do alone   

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [The webstore access log](#the-webstore-access-log)
- [1. How awk works](#1-how-awk-works)
- [2. Built-in variables](#2-built-in-variables)
- [3. Printing fields](#3-printing-fields)
- [4. Pattern matching](#4-pattern-matching)
- [5. Custom field separator](#5-custom-field-separator)
- [6. Conditionals](#6-conditionals)
- [7. Arithmetic and aggregation](#7-arithmetic-and-aggregation)
- [8. BEGIN and END blocks](#8-begin-and-end-blocks)
- [9. awk vs cut — when to use which](#9-awk-vs-cut--when-to-use-which)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

`cut` extracts columns. `grep` finds lines. `sed` transforms content. `awk` does all three at once — and adds arithmetic. It reads a file line by line, splits each line into numbered fields, and lets you filter rows, extract specific fields, compute totals, and format output in a single command. The reason awk exists separately from the other filter tools is calculation — when you need the total bytes transferred across all 200 responses, or the average response time across a thousand requests, awk does it in one line. No spreadsheet. No Python script.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files  ← this file lives here
       awk — field extraction + filtering + arithmetic in one command
  L4  Config  ← webstore.conf — awk reads this with -F '='
  L3  State & Debug  ← /var/log — the logs awk analyses
  L2  Networking
  L1  Process Manager
  L0  Kernel & Hardware
```

awk is the last tool in the text processing chain — grep narrows the lines, cut or awk extracts the fields, awk calculates the numbers. Every incident report you produce from a log file ends with awk.

---

## The webstore access log

This log adds a bytes field compared to the filter commands file. Save it to `~/webstore/logs/access.log`.

```
192.168.1.10 GET /api/products 200 512
192.168.1.11 GET /api/products 200 489
192.168.1.12 POST /api/orders 201 1024
192.168.1.10 GET /api/products 200 512
192.168.1.13 GET /api/users 404 128
192.168.1.14 POST /api/orders 500 256
192.168.1.11 GET /api/products 200 489
192.168.1.15 DELETE /api/orders/7 403 64
192.168.1.10 GET /api/products 200 512
192.168.1.14 POST /api/orders 500 256
```

Fields: `$1`=IP · `$2`=method · `$3`=path · `$4`=status · `$5`=bytes

---

## 1. How awk works

awk reads a file one line at a time. Each line is a **record**. Each record is automatically split into **fields** — by whitespace by default. You write rules:

```
awk 'PATTERN { ACTION }' file
     │         │
     │         └── what to do: print fields, calculate, format
     └──────────── condition to test — if true, action runs
                   if omitted, action runs on every line
```

The simplest awk command:

```bash
awk '{ print }' ~/webstore/logs/access.log
# 192.168.1.10 GET /api/products 200 512
# 192.168.1.11 GET /api/products 200 489
# ...
```

No pattern means match everything. `print` with no arguments prints the whole line (`$0`). Identical to `cat` — but now you understand the structure everything else builds on.

---

## 2. Built-in variables

These are available in every awk program without being defined:

| Variable | What it holds | Value for line `192.168.1.10 GET /api/products 200 512` |
|---|---|---|
| `$0` | The entire current line | `192.168.1.10 GET /api/products 200 512` |
| `$1` | Field 1 | `192.168.1.10` |
| `$2` | Field 2 | `GET` |
| `$3` | Field 3 | `/api/products` |
| `$4` | Field 4 | `200` |
| `$5` | Field 5 | `512` |
| `NR` | Current line number (Number of Record) | `1` on first line, `2` on second |
| `NF` | Number of fields in current line (Number of Fields) | `5` for this log format |
| `FS` | Field separator — default whitespace | Set with `-F` flag or in BEGIN block |

---

## 3. Printing fields

```bash
# Print only the IP address (field 1)
awk '{ print $1 }' ~/webstore/logs/access.log
# 192.168.1.10
# 192.168.1.11
# 192.168.1.12
# ...

# Print IP and status code with default space separator
awk '{ print $1, $4 }' ~/webstore/logs/access.log
# 192.168.1.10 200
# 192.168.1.11 200
# 192.168.1.12 201
# ...

# Print with custom text between fields
awk '{ print $1 " → " $4 }' ~/webstore/logs/access.log
# 192.168.1.10 → 200
# 192.168.1.11 → 200
# ...

# Print line number alongside each line
awk '{ print NR, $0 }' ~/webstore/logs/access.log
# 1 192.168.1.10 GET /api/products 200 512
# 2 192.168.1.11 GET /api/products 200 489
# ...
```

**Comma vs string concatenation:**
`print $1, $4` puts a space between fields (comma = space separator).
`print $1 $4` joins them with nothing between.
`print $1 " → " $4` puts custom text between them.

---

## 4. Pattern matching

A pattern before `{ }` filters which lines trigger the action. Only matching lines run the action block.

```bash
# Print all lines containing "500" anywhere
awk '/500/ { print }' ~/webstore/logs/access.log
# 192.168.1.14 POST /api/orders 500 256
# 192.168.1.14 POST /api/orders 500 256

# Print IP and path for 500 errors only
awk '/500/ { print $1, $3 }' ~/webstore/logs/access.log
# 192.168.1.14 /api/orders
# 192.168.1.14 /api/orders

# Match on a specific field — only lines where field 4 is exactly "500"
awk '$4 == "500" { print }' ~/webstore/logs/access.log
# 192.168.1.14 POST /api/orders 500 256
# 192.168.1.14 POST /api/orders 500 256
```

**`/500/` vs `$4 == "500"` — why it matters:**
`/500/` matches any line containing `500` anywhere — a path like `/api/v500/orders` would match too.
`$4 == "500"` matches only when field 4 is exactly `500`. More precise. Use field matching when you know which column holds the value.

---

## 5. Custom field separator

When your file uses a delimiter other than whitespace, set it with `-F`.

```bash
# webstore.conf uses = as the separator
# db_host=webstore-db
# db_port=5432

# Print only the values (field 2, split on =)
awk -F '=' '{ print $2 }' ~/webstore/config/webstore.conf
# webstore-db
# 5432
# 8080
# webstore-api
# 80
# webstore-frontend
# production

# Print formatted key → value pairs
awk -F '=' '{ print "KEY: " $1 "  VALUE: " $2 }' ~/webstore/config/webstore.conf
# KEY: db_host  VALUE: webstore-db
# KEY: db_port  VALUE: 5432
# ...

# /etc/passwd uses : — print username (field 1) and shell (field 7)
awk -F ':' '{ print $1, $7 }' /etc/passwd
# root /bin/bash
# daemon /usr/sbin/nologin
# akhil /bin/bash
```

---

## 6. Conditionals

`if` inside the action block applies logic beyond simple pattern matching.

```bash
# Print lines where bytes transferred is greater than 500
awk '{ if ($5 > 500) print $1, $3, $5 }' ~/webstore/logs/access.log
# 192.168.1.12 /api/orders 1024
# 192.168.1.10 /api/products 512
# 192.168.1.10 /api/products 512
# 192.168.1.10 /api/products 512

# Print lines where status is NOT 200
awk '{ if ($4 != "200") print $0 }' ~/webstore/logs/access.log
```

The pattern form is more idiomatic awk and reads more cleanly:

```bash
# These two are equivalent — second is preferred
awk '{ if ($4 == "500") print $0 }' access.log
awk '$4 == "500" { print }' access.log
```

Use `if` when the condition is complex or when you need `else`. Use the pattern form for simple single-condition filtering.

---

## 7. Arithmetic and aggregation

This is what separates awk from every other filter tool. Variables persist across lines — you accumulate values as awk reads through the file.

```bash
# Sum total bytes transferred across all requests
awk '{ total += $5 } END { print "Total bytes:", total }' ~/webstore/logs/access.log
# Total bytes: 4242

# Count 500 errors (count++ increments by 1 each time condition matches)
awk '$4 == "500" { count++ } END { print "500 errors:", count }' ~/webstore/logs/access.log
# 500 errors: 2

# Sum bytes for successful requests only
awk '$4 == "200" { total += $5 } END { print "Bytes from 200s:", total }' ~/webstore/logs/access.log
# Bytes from 200s: 2514

# Calculate average bytes per request (NR = total line count at END)
awk '{ total += $5 } END { print "Average bytes:", total/NR }' ~/webstore/logs/access.log
# Average bytes: 424.2
```

**How accumulation works:**
`total += $5` adds field 5 of the current line to `total`. Since `total` starts at zero and this runs on every line, by the time `END` runs, `total` holds the sum of every value in field 5 across the entire file. `count++` works the same way — increments by 1 each time the condition is true.

---

## 8. BEGIN and END blocks

`BEGIN` runs once before any lines are read. `END` runs once after all lines are processed.

```bash
# Full report with header and summary
awk '
  BEGIN { print "--- Webstore Access Report ---" }
  { print $1, $4, $5 }
  END   { print "--- Total requests:", NR, "---" }
' ~/webstore/logs/access.log
# --- Webstore Access Report ---
# 192.168.1.10 200 512
# 192.168.1.11 200 489
# 192.168.1.12 201 1024
# ...
# --- Total requests: 10 ---

# Requests per IP using an associative array
awk '{ count[$1]++ } END { for (ip in count) print count[ip], ip }' \
  ~/webstore/logs/access.log | sort -rn
# 3 192.168.1.10
# 2 192.168.1.14
# 2 192.168.1.11
# 1 192.168.1.15
# 1 192.168.1.13
# 1 192.168.1.12

# Total bytes per status code
awk '{ bytes[$4] += $5 } END { for (s in bytes) print s, bytes[s] }' \
  ~/webstore/logs/access.log | sort
# 200 2514
# 201 1024
# 403 64
# 404 128
# 500 512
```

`count[$1]++` uses an associative array — `$1` (the IP address) is the key, the value increments each time that IP appears. `END` then loops over every key and prints the result.

---

## 9. awk vs cut — when to use which

| Situation | Use |
|---|---|
| Extract one field, simple delimiter | `cut` — faster syntax |
| Extract multiple fields with custom text between them | `awk` |
| Filter rows by the exact value of a field | `awk` |
| Calculate totals, averages, counts | `awk` — `cut` cannot do this |
| Process a config file with `=` or `:` delimiter | Either — `awk -F '='` or `cut -d'='` |
| Build a per-key count or report | `awk` — `cut` has no aggregation |

---

## On the webstore

The webstore has been running. Logs have accumulated.
You need to produce a full incident report from the access log.

```bash
# Step 1 — total requests and total bytes transferred
awk '{ requests++; total += $5 } END { print "Requests:", requests; print "Total bytes:", total }' \
  ~/webstore/logs/access.log
# Requests: 10
# Total bytes: 4242

# Step 2 — count requests per status code
awk '{ count[$4]++ } END { for (s in count) print s, count[s] }' \
  ~/webstore/logs/access.log | sort
# 200 5
# 201 1
# 403 1
# 404 1
# 500 2

# Step 3 — which IPs caused the 500 errors?
awk '$4 == "500" { print $1 }' ~/webstore/logs/access.log | sort | uniq -c | sort -rn
#   2 192.168.1.14

# Step 4 — which endpoints are taking the most bytes?
awk '{ bytes[$3] += $5 } END { for (p in bytes) print bytes[p], p }' \
  ~/webstore/logs/access.log | sort -rn
# 1536 /api/products
# 1024 /api/orders
# 256 /api/orders
# 128 /api/users
# 64 /api/orders/7

# Step 5 — full formatted report
awk '
  BEGIN {
    printf "%-18s %-8s %-25s %-6s %s\n", "IP", "METHOD", "PATH", "STATUS", "BYTES"
    print "----------------------------------------------------------------------"
  }
  { printf "%-18s %-8s %-25s %-6s %s\n", $1, $2, $3, $4, $5 }
  END { print "----------------------------------------------------------------------"; print "Total requests:", NR }
' ~/webstore/logs/access.log
# IP                 METHOD   PATH                      STATUS BYTES
# ----------------------------------------------------------------------
# 192.168.1.10       GET      /api/products             200    512
# ...
# ----------------------------------------------------------------------
# Total requests: 10
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `awk: syntax error` on a pattern | Missing quotes around string comparison — `$4 == 500` vs `$4 == "500"` | String values need quotes: `$4 == "500"`. Numbers do not: `$5 > 500` |
| Field extraction returns nothing | Wrong field number — fields start at `$1` not `$0` | `$0` is the whole line. Fields start at `$1`. Check with `awk '{ print NF }' file` to see field count |
| `-F` not splitting correctly | Delimiter has special meaning in regex | Escape it: `awk -F '\.'` for dot, `awk -F '\|'` for pipe |
| `END` block shows wrong totals | Action ran on header line or blank lines | Filter them out: `NR > 1 { total += $5 }` skips line 1 |
| Associative array output is unsorted | awk arrays have no guaranteed order | Pipe to `sort` after the END block |
| `print $1 $4` joins fields with no space | Missing comma between field references | Use `print $1, $4` (comma = space) or `print $1 " " $4` |

---

## Daily commands

| Command | What it does |
|---|---|
| `awk '{ print $1 }' <file>` | Print field 1 from every line |
| `awk '{ print $1, $4 }' <file>` | Print fields 1 and 4 with space between |
| `awk '$4 == "500" { print }' <file>` | Print lines where field 4 equals exactly 500 |
| `awk -F '=' '{ print $2 }' <file>` | Use `=` as delimiter — print values from config files |
| `awk -F ':' '{ print $1, $7 }' /etc/passwd` | Print username and shell for every user |
| `awk '{ total += $5 } END { print total }' <file>` | Sum all values in field 5 |
| `awk '$4=="500"{ c++ } END{ print c }' <file>` | Count lines where field 4 is 500 |
| `awk '{ total += $5 } END { print total/NR }' <file>` | Average of field 5 across all lines |
| `awk '{ count[$1]++ } END { for (k in count) print count[k], k }' <file>` | Count occurrences per unique value in field 1 |
| `awk 'BEGIN{print "header"} { print } END{print "footer"}' <file>` | Wrap output with header and footer |

---

→ **Interview questions for this topic:** [99-interview-prep → awk](../99-interview-prep/README.md#awk)
