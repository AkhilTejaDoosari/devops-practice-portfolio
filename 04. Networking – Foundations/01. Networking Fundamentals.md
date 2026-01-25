# Networking Fundamentals — From One Server to Cloud, Docker, and Kubernetes
> A chronological “systems grow over time” walkthrough of the networking concepts you actually use.

---

## Overview (What you’ll learn)

You’ll follow one imaginary app (**TravelBody**) as it grows:

- **Single server** → how the world finds you (**IP, DNS**)
- **One server, many apps** → how traffic hits the right process (**Ports**)
- **Security separation** → keep blast radius small (**Subnets, Routing, Firewalls**)
- **Private servers still need internet** → safe outbound access (**NAT**)
- **Cloud version of the same ideas** → managed networking (**VPC, IGW, Route Tables, NAT Gateway**)
- **Containers** → isolated networks + port publishing (**Bridge, Port mapping, Overlay**)
- **Kubernetes** → stable networking for ephemeral workloads (**Pods, Services, Ingress**)

---

## Table of Contents
1. [Single Server: IP + DNS](#1-single-server-ip--dns)
2. [Multiple Apps on One Server: Ports](#2-multiple-apps-on-one-server-ports)
3. [Security & Segmentation: Subnets + Routing + Firewalls](#3-security--segmentation-subnets--routing--firewalls)
4. [Private Subnets Need Outbound: NAT](#4-private-subnets-need-outbound-nat)
5. [Cloud Networking: VPC + Subnets + Gateways + Route Tables](#5-cloud-networking-vpc--subnets--gateways--route-tables)
6. [Container Networking (Docker): Bridge + Port Mapping + Overlay](#6-container-networking-docker-bridge--port-mapping--overlay)
7. [Kubernetes Networking: Pods + Services + Ingress](#7-kubernetes-networking-pods--services--ingress)
8. [Troubleshooting Cheatsheet](#8-troubleshooting-cheatsheet)
---

<details>
<summary><strong>1. Single Server: IP + DNS</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
Users need to **find your server** on the internet.

### IP Address (the identifier)
- Every device on a network needs an address so other devices can send data to it.
- A **public IP** is reachable from the internet.

<p align="center">
  <img src="images/PLACEHOLDER-ip-dns.png" alt="Client resolves DNS and connects to server IP" width="720" />
  <br><em>Figure: Client → DNS → Public IP → Server</em>
</p>

### DNS (name → IP)
- Humans don’t memorize IPs like `203.0.113.10`.
- DNS maps `travelbody.com` → IP address.
- Browser uses that IP to connect.

**Reality check (tiny but important):**
- DNS does not “connect” you. DNS only returns the address.
- Your browser then opens the connection.

</div>
</details>

---

<details>
<summary><strong>2. Multiple Apps on One Server: Ports</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
One server runs multiple apps:
- Website
- Database
- Payment service  
All share one IP. So how does traffic land on the right app?

### Ports (application “doors”)
- Ports are numbers from **1 → 65,535**
- Each app listens on a different port

**Common defaults**
- `80` = HTTP
- `443` = HTTPS
- `3306` = MySQL
- `9090` = custom service

<p align="center">
  <img src="images/PLACEHOLDER-ports.png" alt="One IP with multiple ports mapped to different apps" width="720" />
  <br><em>Figure: One IP address, many ports → many applications</em>
</p>

**Key point**
- `IP + Port` = the exact destination process on that machine.

</div>
</details>

---

<details>
<summary><strong>3. Security & Segmentation: Subnets + Routing + Firewalls</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
If everything is together, one breach gives access to everything.

### Subnets (segmentation)
Split your network into zones:
- **Public-facing frontend**
- **App tier**
- **Database tier** (most restricted)

Example ranges (simple example):
- Frontend subnet: `10.0.1.0/24`
- App subnet: `10.0.2.0/24`
- DB subnet: `10.0.3.0/24`

<p align="center">
  <img src="images/PLACEHOLDER-subnets.png" alt="Network split into frontend, app, db subnets" width="720" />
  <br><em>Figure: Segmenting network into subnets reduces blast radius</em>
</p>

**Rookie mistake (fix this now):**
- A subnet is not “security” by itself.
- Subnet is *structure*. Security comes from rules (firewalls / security groups / ACLs).

### Routing (how traffic moves)
- Routing decides where packets go next.
- It enables: “Frontend can reach DB” (path exists).

### Firewalls (what traffic is allowed)
Routing says: “Can we reach it?”
Firewall says: “Are we allowed?”

Two common layers:
- **Network firewall** (between zones / edge)
- **Host firewall** (on the server)

Typical rules:
- Internet → Frontend: allow `80` and `443`
- Frontend → DB: allow `3306` (only from frontend subnet)
- Deny everything else

<p align="center">
  <img src="images/PLACEHOLDER-firewall-rules.png" alt="Firewall allow-list rules between tiers" width="720" />
  <br><em>Figure: Allow-list between tiers (least privilege)</em>
</p>

</div>
</details>

---

<details>
<summary><strong>4. Private Subnets Need Outbound: NAT</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
Backend servers in private subnets:
- should not be reachable from the internet
- still need outbound access (updates, external APIs)

### NAT (Network Address Translation)
- Private servers keep private IPs like `10.0.2.5`
- NAT lets many private servers share **one public IP** for outbound

**How it behaves**
1. Private server sends outbound request
2. NAT replaces source IP (private → NAT public)
3. Response returns to NAT
4. NAT maps it back to the correct private server

<p align="center">
  <img src="images/PLACEHOLDER-nat.png" alt="Private subnet outbound through NAT to internet" width="720" />
  <br><em>Figure: Private subnet → NAT → Internet (outbound only)</em>
</p>

**Rookie mistake**
- NAT is mainly for outbound internet access.
- NAT does not “publish” private services to the internet.

</div>
</details>

---

<details>
<summary><strong>5. Cloud Networking: VPC + Subnets + Gateways + Route Tables</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
You want the same architecture, but faster scaling and less hardware management.

### VPC (your isolated network in the cloud)
- Your private network boundary (example CIDR: `10.0.0.0/16`)
- Inside it: subnets, routes, security controls

### The BIG blind spot: what makes a subnet public/private?
Not the name. Not your intention.

A subnet becomes:
- **Public** if route table has: `0.0.0.0/0 → Internet Gateway (IGW)`
- **Private** if it does NOT route to IGW (often routes outbound via NAT)

### Internet Gateway (IGW)
- Connects public subnets to the internet

### Route Tables
- Define traffic direction:
  - Public subnet: `0.0.0.0/0 → IGW`
  - Private subnet: `0.0.0.0/0 → NAT Gateway`

### NAT Gateway
- Managed NAT placed in a public subnet
- Private subnet routes outbound traffic through it

<p align="center">
  <img src="images/PLACEHOLDER-vpc-igw-nat.png" alt="VPC with public and private subnets, IGW, NAT GW" width="820" />
  <br><em>Figure: Cloud network = same concepts, managed components</em>
</p>

</div>
</details>

---

<details>
<summary><strong>6. Container Networking (Docker): Bridge + Port Mapping + Overlay</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
You want consistent deployments and fewer “works on my machine” issues.

### Bridge network (single host)
- Docker creates a private network on the host.
- Containers on the same bridge can communicate (often via container names).

### Port mapping (publish container)
Containers have internal ports.
To access from outside the host, you map:

`host_port → container_port`

Example concept:
- Host `:9090` → Container `payment:9090`

<p align="center">
  <img src="images/PLACEHOLDER-docker-port-map.png" alt="Host port mapped to container port" width="780" />
  <br><em>Figure: Host port publishing forwards traffic into the container</em>
</p>

### Overlay network (multi-host)
- Used when containers span multiple servers.
- Provides a virtual network across hosts.

**Rookie mistake**
- Inside container, `localhost` means that container only.

</div>
</details>

---

<details>
<summary><strong>7. Kubernetes Networking: Pods + Services + Ingress</strong></summary>

<div style="margin-left: 16px; margin-right: 16px; margin-top: 8px; margin-bottom: 8px;">

### What problem shows up?
Pods move. Pods die. IPs change. Hardcoding IPs breaks everything.

### Pods
- Basic unit in Kubernetes
- Each pod gets an IP
- Containers inside the pod share that IP

### Services (stable identity)
Pods are ephemeral (temporary). Services fix that:
- stable virtual IP
- stable DNS name (`database-service`)
- forwards traffic to healthy pods behind it

<p align="center">
  <img src="images/PLACEHOLDER-k8s-service.png" alt="Kubernetes service routes to pod replicas" width="820" />
  <br><em>Figure: Service gives stable access even when pods change</em>
</p>

### Ingress (HTTP routing into the cluster)
Ingress routes external HTTP/HTTPS traffic to internal services based on rules:
- `/` → website service
- `/api/booking` → booking service
- `/api/payment` → payment service

<p align="center">
  <img src="images/PLACEHOLDER-k8s-ingress.png" alt="Ingress routes paths to services" width="820" />
  <br><em>Figure: One entry point, many internal services</em>
</p>

**Important**
- Ingress requires an Ingress Controller (NGINX Ingress etc.) to actually work.

</div>
</details>

---

<details>
<summary><strong>8. Troubleshooting Cheatsheet</strong></summary>

| Symptom                                           | Usually Means                          | Check                                               |  
|---------------------------------------------------|----------------------------------------|-----------------------------------------------------|  
| Domain doesn’t work                               | DNS not resolving correctly            | DNS record, correct target, propagation             |  
| IP works but site doesn’t                         | wrong port / blocked port              | firewall/security group, app listening on 80/443    |  
| Frontend can’t reach DB                           | routing exists but blocked OR no route | route tables + firewall rules for DB port           |  
| Private instances can’t download updates          | no NAT path outbound                   | private route table → NAT, NAT exists, egress rules |  
| Docker container works internally but not outside | port not published                     | host port mapping + host firewall                   |  
| K8s app breaks on redeploy                        | pod IP changed                         | use Service DNS, not pod IP                         |  
| Ingress routes nowhere                            | controller missing or wrong rules      | ingress controller, service ports, rules            |  

</details>

---