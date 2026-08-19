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

# Linux Networking

> **Layer:** L2 — Networking   
> **Depends on:** [12 Service Management](../12-service-management/README.md) — you need running services before you have network traffic to debug   
> **Used in production when:** nginx is running but not responding, the API cannot reach the database, a port that should be open is not, or you need to trace where a request is failing.  

---

## Table of Contents

- [What this is](#what-this-is)
- [How it fits the stack](#how-it-fits-the-stack)
- [1. ip — inspect network interfaces](#1-ip--inspect-network-interfaces)
- [2. ping — confirm reachability](#2-ping--confirm-reachability)
- [3. traceroute — find where delay lives](#3-traceroute--find-where-delay-lives)
- [4. dig — query DNS](#4-dig--query-dns)
- [5. curl — test HTTP endpoints](#5-curl--test-http-endpoints)
- [6. ss — see what is listening](#6-ss--see-what-is-listening)
- [7. nc — test port connectivity](#7-nc--test-port-connectivity)
- [8. tcpdump — capture live traffic](#8-tcpdump--capture-live-traffic)
- [9. nmap — scan open ports](#9-nmap--scan-open-ports)
- [On the webstore](#on-the-webstore)
- [What breaks](#what-breaks)
- [Daily commands](#daily-commands)

---

## What this is

When something is wrong with a running service, the problem is often in the network layer. nginx is running but not responding. The API cannot reach the database. A port that should be open is not. A request arrives but takes 3 seconds and you do not know where the delay is. These tools answer those questions from the command line — no GUI, no external monitoring tool, just the terminal and the commands that show you exactly what is happening on the network right now.

---

## How it fits the stack

```
  L6  You
  L5  Tools & Files
  L4  Config  ← /etc/hosts /etc/netplan — network config lives here
  L3  State & Debug  ← /proc/net /sys/class/net — live network state
  L2  Networking  ← this file lives here
       ip ping traceroute dig curl ss nc tcpdump nmap
  L1  Process Manager
  L0  Kernel & Hardware  ← TCP/IP stack is in the kernel
```

L2 sits between the kernel's TCP/IP stack (L0) and the config that shapes it (L4). Every service at L1 that listens on a port is visible through the tools at L2.

---

## 1. ip — inspect network interfaces

`ip` shows and configures network interfaces. When you SSH into a server for the first time, `ip addr` tells you what IP addresses the machine has.

```bash
# Show all interfaces and their IP addresses
ip addr show
# 1: lo: <LOOPBACK,UP,LOWER_UP>
#     inet 127.0.0.1/8 scope host lo
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.0.1.45/24 brd 10.0.1.255 scope global eth0
```

`lo` is the loopback interface — `127.0.0.1`. `eth0` (or `enp3s0` on newer systems) is the real network interface.

```bash
# Show the routing table — how the server decides where to send traffic
ip route show
# default via 10.0.1.1 dev eth0   ← default gateway
# 10.0.1.0/24 dev eth0 proto kernel   ← local network

# Show a specific interface
ip addr show eth0
```

---

## 2. ping — confirm reachability

`ping` sends ICMP echo requests and measures whether a host responds.

```bash
# Ping the webstore API — stop after 4 packets (-c = --count)
ping -c 4 webstore-api
# 64 bytes from 172.18.0.3: icmp_seq=0 ttl=64 time=0.312 ms
# 4 packets transmitted, 4 received, 0% packet loss

# Ping the database
ping -c 3 webstore-db

# Ping localhost — confirm loopback is up
ping -c 2 localhost
```

`time=0.312 ms` is round-trip latency. Under 1ms on a local network is normal. Packet loss above 0% means something is dropping packets. A failed ping does not always mean the host is down — some servers block ICMP. Follow up with `nc` to test a specific port.

---

## 3. traceroute — find where delay lives

`traceroute` maps every router hop between you and a destination, showing latency at each step.

```bash
# Trace path to API server
traceroute webstore-api.example.com

# Skip DNS lookups — faster, IPs only (-n = numeric)
traceroute -n webstore-api.example.com
#  1  10.0.1.1     0.891 ms   ← your gateway
#  2  172.16.0.1   1.234 ms   ← ISP router
#  3  54.239.1.1   8.456 ms   ← AWS edge
#  4  54.239.2.15  10.123 ms  ← destination
```

Each line is one hop. `* * *` means a router is blocking traceroute probes — not necessarily broken. Use when API response times jumped and you need to find which hop is adding the latency.

---

## 4. dig — query DNS

`dig` (Domain Information Groper) queries DNS servers and shows the full response. Use when a hostname is not resolving or resolving to the wrong IP.

```bash
# Quick IP lookup
dig +short webstore-api.example.com
# 54.239.28.81

# Query a specific DNS server — bypass your default resolver
dig @8.8.8.8 webstore-api.example.com

# Trace full DNS resolution from root servers down
dig +trace webstore-api.example.com

# Look up the nameserver for a domain (NS record)
dig webstore-api.example.com NS

# Check TTL — how long until this record expires from cache
dig webstore-api.example.com
# webstore-api.example.com.  300  IN  A  54.239.28.81
#                            ^^^
#                            TTL in seconds — 300 = 5 minutes
```

TTL matters when you update a DNS record and it is not working yet — the old answer is cached until TTL expires.

---

## 5. curl — test HTTP endpoints

`curl` makes HTTP requests from the terminal. Essential on a server with no GUI.

```bash
# Test if the webstore API responds
curl http://localhost:8080

# Check only the response status code and headers (-I = --head)
curl -I http://localhost:8080/api/products
# HTTP/1.1 200 OK
# Content-Type: application/json

# Verbose — see full request and response headers (-v = --verbose)
curl -v http://localhost:8080/api/products

# POST request with JSON body
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": 1, "quantity": 2}'

# Follow redirects (-L = --location)
curl -L http://webstore.example.com

# Fail fast — give up after 5 seconds (--max-time)
curl --max-time 5 http://localhost:8080/api/products

# Test virtual host routing with custom Host header (-H = --header)
curl -H "Host: webstore.example.com" http://localhost
```

**Reading status codes:** `200` = success, `301/302` = redirect, `404` = not found, `502 Bad Gateway` = nginx reached but upstream API did not respond, `503` = nginx could not reach upstream at all.

---

## 6. ss — see what is listening

`ss` (Socket Statistics) shows every active connection and every port the server is listening on. Replaced `netstat` on modern Linux.

```bash
# Show all listening TCP ports with process names
# -t = TCP, -l = listening, -n = numeric, -p = process name
sudo ss -tlnp
# LISTEN  0  511  0.0.0.0:80    users:(("nginx",pid=1235))
# LISTEN  0  128  0.0.0.0:22    users:(("sshd",pid=845))
# LISTEN  0  128  127.0.0.1:5432 users:(("postgres",pid=987))
```

Port 80 nginx on `0.0.0.0` = accessible from outside.
Port 5432 postgres on `127.0.0.1` = local only, not exposed externally. Good.

```bash
# Show all TCP and UDP connections with process names
# -u = UDP, -t = TCP, -n = numeric, -p = process
sudo ss -tunp

# Check if nginx is on port 80
sudo ss -tlnp | grep :80

# Show established connections only
sudo ss -t state established
```

---

## 7. nc — test port connectivity

`nc` (netcat) opens a raw TCP connection to a port — the fastest way to test whether a specific port is open without speaking the full protocol.

```bash
# Test if API port 8080 is accepting connections
# -z = zero I/O (just test), -v = verbose
nc -zv webstore-api 8080
# Connection to webstore-api 8080 port [tcp/*] succeeded!

# Test database port
nc -zv webstore-db 5432

# Test with a timeout — fail after 3 seconds (-w = wait)
nc -zv -w 3 webstore-api 8080
```

If `nc` fails, it is a network or firewall problem. If `nc` succeeds but the application still cannot connect, the problem is in the application layer — wrong credentials, wrong database name, wrong connection string.

---

## 8. tcpdump — capture live traffic

`tcpdump` captures raw network packets in real time. The deepest debugging tool — reach for it when everything else has failed to explain what is happening.

```bash
# Capture all traffic on eth0 — Ctrl+C to stop
sudo tcpdump -i eth0

# Capture only HTTP traffic on port 80
sudo tcpdump -i eth0 port 80

# Capture traffic to/from a specific host
sudo tcpdump -i eth0 host 10.0.1.45

# No DNS lookups — show IPs only (-n = numeric)
sudo tcpdump -i eth0 -n port 80

# Show packet contents in ASCII (-A = ASCII)
sudo tcpdump -i eth0 -A port 8080

# Save to file for analysis later (-w = write)
sudo tcpdump -i eth0 -w capture.pcap port 8080

# Read a saved capture file (-r = read)
sudo tcpdump -r capture.pcap
```

`-A port 8080` shows you the raw HTTP request and response — every header and body. Use when `curl` returns something unexpected and you need to see exactly what is on the wire.

---

## 9. nmap — scan open ports

`nmap` probes a host and reports which ports are open. On your own servers, use it to confirm your firewall is configured correctly.

```bash
# Scan the webstore server
nmap webstore.example.com

# Scan specific ports only (-p = ports)
nmap -p 22,80,443,8080 webstore.example.com

# Fast scan — top 100 ports (-F = fast)
nmap -F webstore.example.com

# Output:
# PORT     STATE   SERVICE
# 22/tcp   open    ssh
# 80/tcp   open    http
# 5432/tcp closed  postgresql   ← good — DB should not be exposed
```

Run `nmap` from an external machine to get the attacker's view of your server — what they can see.

---

## On the webstore

Users report the webstore is not loading. Work from outside in.

```bash
# Step 1 — is nginx running and bound to port 80?
sudo ss -tlnp | grep :80
# Nothing? nginx is not listening
sudo systemctl status nginx
journalctl -u nginx -n 20

# Step 2 — can the server respond to HTTP at all?
curl -I http://localhost
# 200 OK → nginx is up
# Connection refused → nginx not running or not on port 80

# Step 3 — can the API port be reached from the frontend server?
nc -zv webstore-api 8080
# succeeded → network is fine
# failed → check if API service is running, check firewall

# Step 4 — is the API responding correctly?
curl -v http://webstore-api:8080/api/products

# Step 5 — can the API reach the database?
nc -zv webstore-db 5432
# succeeded → DB port reachable
# failed → DB is down or firewall is blocking

# Step 6 — is DNS resolving to the right IP?
dig +short webstore-api.example.com
# compare to the IP you expect

# Step 7 — traffic arriving but responses wrong? capture it
sudo tcpdump -A -i eth0 port 8080 -c 20
# read the raw HTTP request and response
```

---

## What breaks

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (7) Failed to connect` | Service not running or wrong port | `ss -tlnp` to check what is listening, `systemctl status` to check service |
| `curl` returns `502 Bad Gateway` | nginx is up but upstream API is not responding | `nc -zv api-host 8080` to test API port, `journalctl -u api-service` |
| `ping` succeeds but `nc` fails | ICMP allowed but the specific port is blocked by firewall | Check `ufw status` or `iptables -L` — the port may be firewalled |
| `dig +short` returns old IP after DNS update | TTL has not expired — old answer is still cached | Wait for TTL to expire, or `dig @8.8.8.8` to check what authoritative DNS has |
| `ss -tlnp` shows service on `127.0.0.1` not `0.0.0.0` | Service is bound to localhost only — not accessible from outside | Edit service config to bind to `0.0.0.0` or the correct interface |
| `tcpdump` shows packets arriving but no response | Service is receiving but not responding — likely a crash or busy | `journalctl -u service -f` while sending a request to see the error |

---

## Daily commands

| Command | What it does |
|---|---|
| `ip addr show` | Show all interfaces and IP addresses |
| `ip route show` | Show routing table and default gateway |
| `ping -c 4 <host>` | Test if a host is reachable |
| `dig +short <host>` | Quick DNS lookup — returns just the IP |
| `curl -I <url>` | Check HTTP status code and headers only |
| `curl -v <url>` | Full verbose HTTP request and response |
| `sudo ss -tlnp` | Show all listening ports with process names |
| `nc -zv <host> <port>` | Test if a specific port is open |
| `sudo tcpdump -A -i eth0 port <port>` | Capture and read raw traffic on a port |
| `nmap -p <ports> <host>` | Scan specific ports from outside |

---

→ **Interview questions for this topic:** [99-interview-prep → Networking](../99-interview-prep/README.md#networking)
