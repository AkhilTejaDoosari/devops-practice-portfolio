[Home](../README.md) | [GHA Fundamentals](../01-ci-layer/01-actions-fundamentals.md) | [GitOps Bridge](../01-ci-layer/02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](../02-cd-layer/00-lab-argocd-setup.md) | [ArgoCD Ops](../02-cd-layer/01-argocd-operations.md) | [Trivy Scan](01-trivy-container-scanning.md) | [Interview Prep](../99-interview-prep/README.md)

---

# Trivy — Container Security Scanning

> **Used in production when:** a developer pushes a code change, the pipeline builds a new Docker image, and before that image is allowed anywhere near Docker Hub or your cluster, Trivy scans every package inside it and blocks the push if it finds a critical known vulnerability.

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [What a CVE is](#what-a-cve-is)
- [How Docker images carry vulnerabilities](#how-docker-images-carry-vulnerabilities)
- [Where Trivy sits in the pipeline](#where-trivy-sits-in-the-pipeline)
- [The Trivy step — every line explained](#the-trivy-step--every-line-explained)
- [Reading a Trivy report](#reading-a-trivy-report)
- [How to fix a CVE](#how-to-fix-a-cve)
- [ShopStack CVEs found and fixed](#shopstack-cves-found-and-fixed)
- [What breaks and why](#what-breaks-and-why)
- [Daily commands](#daily-commands)

---

## Why this exists

When you write a Dockerfile, the first line is almost always `FROM something`. That something is not empty — it is an entire operating system with hundreds of packages already installed. You did not write those packages. You did not choose their versions. You inherited them by choosing that base image.

```
FROM nginx:1.24-alpine
```

What you just pulled in:
- Alpine Linux 3.17 (the OS)
- nginx 1.24 (the web server)
- libexpat (an XML parsing library)
- libssl, libcrypto (OpenSSL — handles encryption)
- dozens of other system libraries

Your Nginx config and your HTML files are maybe 10 files. The rest of the image — the part you did not write — is hundreds of packages. Any of those packages could have a known security vulnerability.

Without Trivy, you would not know. You would push the image. ArgoCD would deploy it. It would run in production. And an attacker who checks the public CVE database — which anyone can do — would know exactly which version of libexpat is running in your cluster and exactly how to exploit it.

Trivy fixes this by scanning the image before it ever reaches Docker Hub.

---

## What a CVE is

CVE stands for Common Vulnerabilities and Exposures. It is a public record of a known security bug.

Every CVE has:
- **An ID** — e.g. `CVE-2024-45491` — a unique reference number
- **A severity** — CRITICAL, HIGH, MEDIUM, or LOW
- **A description** — what the bug is and what an attacker can do with it
- **A status** — whether a fix has been released yet

The CVE database is public. Anyone can read it. When a security researcher finds a bug in libexpat, they publish a CVE. The package maintainers release a patched version. The CVE record is updated with the fixed version number.

Trivy downloads this database and checks every package in your image against it. If your image has libexpat version 2.6.2 and the database says 2.6.2 has CVE-2024-45491, Trivy flags it.

Severity levels:
```
CRITICAL  →  Active exploits exist in the wild. Real attackers are using this right now.
             Fix immediately. Do not ship this image.

HIGH      →  Serious vulnerability. A fix exists. Should be resolved soon.

MEDIUM    →  Moderate risk. Usually requires specific conditions to exploit.

LOW       →  Minimal risk. Track it but unlikely to block deployment.
```

In ShopStack, Trivy is configured to only block on CRITICAL. This is deliberate — blocking on MEDIUM or HIGH would generate so much noise that the pipeline would never pass.

---

## How Docker images carry vulnerabilities

Understanding this is what makes Trivy click.

When you run `docker build`, Docker processes each line of your Dockerfile and creates a layer.

```
FROM nginx:1.24-alpine         ← layer 0: the entire nginx+alpine OS (hundreds of packages)
RUN rm /etc/nginx/conf.d/...   ← layer 1: removes a file from the OS layer
COPY nginx.conf ...            ← layer 2: adds your config
COPY html/ ...                 ← layer 3: adds your HTML files
EXPOSE 80                      ← metadata, not a layer
```

The vulnerability is in layer 0 — the base image. You did not write layer 0. You inherited it. But your image contains it. When someone pulls `akhiltejadoosari/shopstack-frontend:b5dcd64` and runs it, they are running layer 0, which contains the vulnerable libexpat.

This is why fixing a CVE almost never means changing your application code. It means changing the base image — swapping `FROM nginx:1.24-alpine` for `FROM nginx:1.30.2-alpine3.23` — because the newer base image ships with a patched version of libexpat.

---

## Where Trivy sits in the pipeline

Before Trivy was added, the build-and-push job in each workflow looked like this:

```
checkout → set SHA tag → login → BUILD AND PUSH (one step) → update manifest
```

The image was built and pushed to Docker Hub in a single step. By the time you could scan it, it was already in the registry. ArgoCD could deploy it before any scan results came back.

After Trivy was added:

```
checkout → set SHA tag → login → BUILD (no push) → TRIVY SCAN → PUSH → update manifest
```

The build and push are now two separate steps. Between them sits Trivy.

```
Build image (push: false, load: true)
    ↓
Image exists locally on the runner — NOT in Docker Hub, NOT visible to ArgoCD
    ↓
Trivy scans the local image
    ↓
          CRITICAL CVE with a fix exists?
          ├── YES → exit code 1 → step fails → pipeline stops
          │         push step never runs → image never reaches Docker Hub
          │         ArgoCD sees no manifest change → cluster unchanged
          │         you fix the base image → push a new Dockerfile → pipeline reruns
          │
          └── NO  → exit code 0 → push step runs
                    image reaches Docker Hub
                    update-manifest job runs
                    ArgoCD syncs cluster
                    clean image deployed
```

The key is `push: false` combined with `load: true`.

- `push: false` — build the image but do not push it anywhere
- `load: true` — load the built image into the runner's local Docker daemon

Without `load: true`, the image is built in a temporary build context and discarded. Trivy cannot see it. With `load: true`, the image sits in the local Docker daemon where Trivy can reference it by name and scan it.

---

## The Trivy step — every line explained

This is what the Trivy step looks like in each ShopStack workflow:

```yaml
- name: Build API image
  uses: docker/build-push-action@v5
  with:
    context: ./services/api
    push: false       # build locally — do NOT push to Docker Hub
    tags: akhiltejadoosari/shopstack-api:${{ steps.tag.outputs.sha }}
    load: true        # load into local Docker daemon so Trivy can scan it

- name: Trivy scan — API image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: akhiltejadoosari/shopstack-api:${{ steps.tag.outputs.sha }}
    # ^ the image to scan — same tag that was just built and loaded locally
    # Trivy looks this up in the local Docker daemon, not Docker Hub

    format: table
    # ^ how to display the results in the Actions log
    # table = human-readable rows and columns
    # other options: json (for scripts), sarif (for GitHub Security tab)

    exit-code: 1
    # ^ if vulnerabilities are found at the specified severity, exit with code 1
    # GitHub Actions treats any non-zero exit code as a step failure
    # step failure = job failure = pipeline stops = push step never runs
    # without this setting: Trivy prints the report and exits 0 regardless of findings
    # the pipeline would pass even with critical CVEs

    severity: CRITICAL
    # ^ only exit with code 1 if CRITICAL vulnerabilities are found
    # why not HIGH too?
    # a typical base image has dozens of HIGH vulnerabilities
    # most have no fix, most require specific conditions to exploit
    # blocking on HIGH would mean the pipeline almost never passes
    # CRITICAL = active exploits, real attackers, real damage — worth blocking for

    ignore-unfixed: true
    # ^ skip CVEs where no fix has been released yet
    # if a CVE exists but the package maintainers have not patched it yet
    # there is nothing you can do — you cannot fix what has no fix
    # blocking the pipeline on an unfixable CVE stops all deployments for no benefit
    # when a fix is released, Trivy will catch it automatically on the next scan

- name: Push API image
  run: docker push akhiltejadoosari/shopstack-api:${{ steps.tag.outputs.sha }}
  # ^ this step only runs if the Trivy step exited with code 0 (passed)
  # if Trivy found a CRITICAL CVE and exited 1, this step is skipped entirely
  # the image never reaches Docker Hub
```

---

## Reading a Trivy report

When a scan fails, open the failing step in GitHub Actions. The log will contain a table like this:

```
Report Summary
┌─────────────────────────────────────────────┬────────┬─────────────────┐
│ Target                                      │ Type   │ Vulnerabilities │
├─────────────────────────────────────────────┼────────┼─────────────────┤
│ shopstack-frontend:b5dcd64 (alpine 3.21.3)  │ alpine │        2        │
└─────────────────────────────────────────────┴────────┴─────────────────┘
```

The summary tells you:
- which image was scanned (`shopstack-frontend:b5dcd64`)
- which OS it detected (`alpine 3.21.3`)
- how many vulnerabilities were found (`2`)

Then the detailed table:

```
Total: 2 (CRITICAL: 2)

┌────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┐
│  Library   │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │
├────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┤
│ libcrypto3 │ CVE-2026-31789 │ CRITICAL │ fixed  │ 3.3.3-r0          │ 3.3.7-r0      │
├────────────┤                │          │        │                   │               │
│ libssl3    │                │          │        │                   │               │
└────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┘
```

How to read each column:

```
Library           = libcrypto3, libssl3
                    the specific package inside your image that has the bug
                    you did not install this — it came from the base image

Vulnerability     = CVE-2026-31789
                    the public CVE ID — search this on nvd.nist.gov to read the full description

Severity          = CRITICAL
                    this is why the pipeline failed — you configured exit-code: 1 on CRITICAL

Status            = fixed
                    a patched version exists — you CAN fix this
                    if it said "affected" — no patch exists yet, ignore-unfixed: true would skip it

Installed Version = 3.3.3-r0
                    this is the version currently in your image

Fixed Version     = 3.3.7-r0
                    this is the version that contains the fix
                    your goal: get your image to use 3.3.7-r0 or newer
```

The fix is almost never to install the patched package directly. It is to upgrade the base image to a version that already ships the patched package.

---

## How to fix a CVE

### Pattern 1 — CVE in an Alpine OS package

Most CVEs in ShopStack's nginx and frontend images are in Alpine OS packages — libexpat, libssl, libcrypto. The Alpine OS version determines which package versions are included.

When Trivy finds a CVE in an Alpine package:
1. Find which Alpine version your current base image uses
2. Find a newer base image that uses an Alpine version with the patched package
3. Update `FROM` in the Dockerfile
4. Push the change — pipeline re-triggers — Trivy scans the new image

```dockerfile
# shopstack-frontend Dockerfile

# BEFORE — nginx:1.24-alpine uses Alpine 3.17.7
# Alpine 3.17 ships libexpat 2.6.2-r0 which has CVE-2024-45491 and CVE-2024-45492
FROM nginx:1.24-alpine

# AFTER — nginx:1.30.2-alpine3.23 uses Alpine 3.23
# Alpine 3.23 ships libexpat 2.6.3-r0 — both CVEs patched
FROM nginx:1.30.2-alpine3.23
```

### Pattern 2 — CVE in a language runtime

ShopStack's worker image uses Go. When the Go standard library itself has a CVE — not an OS package, but the Go runtime — the fix is to upgrade the Go version in the builder stage.

```dockerfile
# shopstack-worker Dockerfile

# BEFORE — Go 1.22 stdlib has CVE-2025-68121 in the TLS implementation
FROM golang:1.22-alpine AS builder

# AFTER — Go 1.24 has the fix applied to stdlib
FROM golang:1.24-alpine AS builder
```

### Pattern 3 — CVE with no fix

If the Trivy report shows `Status: affected` instead of `Status: fixed`, no patch exists yet. `ignore-unfixed: true` already handles this — Trivy skips these CVEs and does not count them toward the exit code. The pipeline passes. You check back when a new version of the package is released.

### The rule — never use `latest`

```dockerfile
# Bad — unpinned, changes without warning
FROM nginx:latest
FROM nginx:alpine

# Good — pinned to a specific version
FROM nginx:1.30.2-alpine3.23
FROM golang:1.24-alpine
```

Using `latest` means your image changes every time you build it, even without touching your Dockerfile. A new CVE in a newer nginx version would break your pipeline without you changing anything. With a pinned version, you control exactly when your base image changes.

---

## ShopStack CVEs found and fixed

These are real CVEs that were in ShopStack images before Trivy was added.

### shopstack-frontend — Run 1 — nginx:1.24-alpine (Alpine 3.17.7)

Trivy found:

```
Library   CVE             Severity  Status  Installed  Fixed
libexpat  CVE-2024-45491  CRITICAL  fixed   2.6.2-r0   2.6.3-r0
libexpat  CVE-2024-45492  CRITICAL  fixed   2.6.2-r0   2.6.3-r0
```

What these CVEs are: integer overflow vulnerabilities in libexpat — the XML parsing library. An attacker can craft a malicious XML document and cause the parser to overflow, potentially crashing the process or executing arbitrary code.

Fix: updated Dockerfile to `FROM nginx:1.27-alpine`

### shopstack-frontend — Run 2 — nginx:1.27-alpine (Alpine 3.21.3)

Trivy found — a new CVE on a newer image:

```
Library    CVE             Severity  Status  Installed  Fixed
libcrypto3 CVE-2026-31789  CRITICAL  fixed   3.3.3-r0   3.3.7-r0
libssl3    CVE-2026-31789  CRITICAL  fixed   3.3.3-r0   3.3.7-r0
```

What this CVE is: a heap buffer overflow in OpenSSL affecting 32-bit systems processing large X.509 certificates. CRITICAL because OpenSSL handles TLS — the encryption layer for all HTTPS traffic.

Fix: updated Dockerfile to `FROM nginx:1.30.2-alpine3.23` (Alpine 3.23 ships OpenSSL 3.3.7-r0)

### shopstack-frontend — Run 3 — nginx:1.30.2-alpine3.23

Trivy found: no CRITICAL CVEs. Pipeline passed. Image pushed.

### shopstack-worker — Run 1 — golang:1.22-alpine (Alpine 3.19.9)

Trivy found:

```
Library  CVE             Severity  Status  Installed  Fixed
stdlib   CVE-2025-68121  CRITICAL  fixed   v1.22.12   1.24.13
```

Note: this CVE is not in an Alpine package. It is in `stdlib` — the Go standard library that was compiled into the worker binary itself. The binary is `app/worker` and it was compiled using Go 1.22.12.

What this CVE is: incorrect certificate validation during TLS session resumption. A man-in-the-middle attacker can forge certificates during a resumed TLS session. CRITICAL because TLS is what makes network communication secure.

Fix: updated Dockerfile builder stage to `FROM golang:1.24-alpine` — the worker binary is now compiled with Go 1.24 which has the fix.

### shopstack-worker — Run 2 — golang:1.24-alpine

Trivy found: no CRITICAL CVEs. Pipeline passed. Image pushed.

---

## What breaks and why

### Pipeline fails — CRITICAL CVE found

```
Symptom:  Trivy step shows red X — "Error: Process completed with exit code 1"
          Trivy report shows one or more CRITICAL rows with Status: fixed
          Push step is skipped — image not in Docker Hub

Cause:    Base image contains a patched CRITICAL CVE
          Trivy found it, exit-code: 1 triggered, step failed

Fix:
  1. Read the Trivy report — find the Library, CVE ID, and Fixed Version
  2. Identify the base image that carries the vulnerable package
     - Alpine OS packages (libexpat, openssl, curl) → upgrade the Alpine base image
     - Language runtime (Go stdlib, Python, Node) → upgrade the language version
  3. Update the FROM line in the Dockerfile
  4. Push the Dockerfile change → pipeline re-triggers → Trivy scans → verify passes
```

### Pipeline passes but CVEs still present (wrong config)

```
Symptom:  Trivy step is green — but you see CVE rows in the log
          Image was pushed with known vulnerabilities

Cause:    exit-code: 1 is missing or set to 0
          Without exit-code: 1, Trivy prints the report but exits 0 regardless
          Pipeline continues and pushes the vulnerable image

Fix:
  Add exit-code: 1 to the Trivy step configuration
  Verify it is under the `with:` block, not under `env:` or elsewhere
```

### Trivy step fails — image not found

```
Symptom:  Trivy step fails with "image not found" or similar error
          Not a CVE — Trivy cannot even find the image to scan

Cause:    load: true is missing from the build step
          Without load: true, the image is built in a temporary build context
          It is not loaded into the local Docker daemon
          Trivy looks in the local Docker daemon and finds nothing

Fix:
  Add load: true to the docker/build-push-action step
  Must be combined with push: false
  Correct build step:
    push: false
    load: true
```

### Trivy DB download fails — transient error

```
Symptom:  Trivy step fails with "failed to download vulnerability DB"
          or a timeout/network error

Cause:    Trivy downloads the CVE database from GitHub releases on each run
          (unless cached) — transient network issue on the runner

Fix:
  Re-run the job — GitHub Actions UI → click the failed run → Re-run failed jobs
  This is a transient runner issue, not a code problem
  The CVE database is cached after the first download — subsequent runs are faster
```

### CVE appears in language binary, not OS packages

```
Symptom:  Trivy report shows a CVE in "gobinary" type, not "alpine" type
          The Library column shows "stdlib" instead of an Alpine package name

Cause:    The compiled binary itself was built with a vulnerable language version
          This is different from OS packages — upgrading the Alpine base alone won't fix it
          The builder stage uses an old Go/Python/Node version that compiled the bug in

Fix:
  Update the builder stage FROM in the Dockerfile
  FROM golang:1.22-alpine AS builder → FROM golang:1.24-alpine AS builder
  The binary is recompiled with the patched stdlib — CVE no longer present in the binary
```

---

## Daily commands

There are no Trivy commands you run manually on your Mac or EC2. Trivy runs automatically inside the GitHub Actions pipeline. Your interaction with it is:

**When a scan fails:**

```
1. Go to github.com/AkhilTejaDoosari/shopstack → Actions tab
2. Click the failed pipeline run
3. Click the failed job (build-and-push)
4. Click the "Trivy scan" step
5. Read the report table — find Library, Installed Version, Fixed Version
6. Open your Dockerfile — change the FROM line
7. Push the Dockerfile change
8. Watch the pipeline rerun
```

**Checking which image version is currently running in the cluster:**

```bash
# On EC2 — check what image tag is on each pod
kubectl describe pod -l tier=frontend | grep Image
kubectl describe pod -l tier=worker | grep Image
kubectl describe pod -l tier=api | grep Image

# Check what tag is in the manifest
grep "image:" infra/k8s/frontend-deployment.yaml
grep "image:" infra/k8s/worker-deployment.yaml
grep "image:" infra/k8s/api-deployment.yaml
```

**Finding the right pinned image version:**

```
hub.docker.com → search the image name → Tags tab → find the latest stable version

Pinning rules:
  nginx:1.30.2-alpine3.23     ← specific nginx version on specific Alpine — best
  nginx:1.30-alpine           ← specific nginx major.minor, any Alpine patch — acceptable
  nginx:alpine                ← any nginx on Alpine — bad, unpredictable
  nginx:latest                ← worst — never use this

When a new CVE appears:
  Check hub.docker.com for the latest patched version
  Update FROM in Dockerfile
  Push — Trivy will confirm the CVE is gone
```
