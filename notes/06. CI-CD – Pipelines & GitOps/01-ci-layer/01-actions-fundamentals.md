[Home](../README.md) | [GHA Fundamentals](01-actions-fundamentals.md) | [GitOps Bridge](02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](../02-cd-layer/00-lab-argocd-setup.md) | [ArgoCD Ops](../02-cd-layer/01-argocd-operations.md) | [Trivy Scan](../03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](../99-interview-prep/README.md)

# GitHub Actions — Fundamentals & Architecture

> **CI vs CD:** GitHub Actions handles Continuous Integration (CI) — it builds the image, tests it, and pushes it to Docker Hub. ArgoCD handles Continuous Deployment (CD) — it watches the repo and syncs the cluster. **GitHub Actions builds. ArgoCD deploys. They are two separate systems**.

---

## Table of Contents
- [1. The Runner Mental Model](#1-the-runner-mental-model-most-important)
- [2. Workflow File Anatomy](#2-workflow-file-anatomy)
- [3. Secrets & Authentication](#3-secrets--authentication)
- [4. Tracing Builds with Tags](#4-tracing-builds-with-tags)

---

## 1. The Runner Mental Model (Most Important)

The runner is a disposable Linux VM. It starts blank. No files, no installed packages beyond base Ubuntu, no credentials, and no memory of previous runs. It runs your steps top to bottom, and when it finishes, it is destroyed permanently.

```text
Run starts
    Blank Ubuntu VM spins up
    Has: bash, curl, git, basic Linux tools
    Has NOT: your code, your secrets, Docker Hub auth
          ↓
Step 1: actions/checkout
    → clones your GitHub repo onto the runner
    → without this step, the runner has zero files
          ↓
Step 2: docker/login-action
    → authenticates to Docker Hub using secrets
          ↓
Step 3: docker/build-push-action
    → builds the image and pushes to Docker Hub
          ↓
Job completes → VM destroyed → gone forever

```

* **Why `actions/checkout` is always first:** The runner is a blank machine. You must clone the repo before you can do anything with it.

---

## 2. Workflow File Anatomy

A workflow is a YAML file that lives inside your repo at `.github/workflows/`. It is version controlled just like application code.

### The Four Building Blocks:

1. **Trigger (`on:`):** What event starts the workflow.
2. **Runner (`runs-on:`):** What machine runs the job.
3. **Job (`jobs:`):** A group of steps that run together on one runner.
4. **Step (`- name:`):** One action inside a job running in sequence, top to bottom.

**How they look in code:**

```yaml
on:                                  # 1. TRIGGER
  push:
    branches: [main]

jobs:                                # 3. JOB
  build-and-push:
    runs-on: ubuntu-latest           # 2. RUNNER
    
    steps:                           # 4. STEPS
      - name: Checkout code
        uses: actions/checkout@v4

```

---

## 3. Secrets & Authentication

You cannot put your passwords in the YAML file because it is public. Secrets live in GitHub Settings and are injected as environment variables at runtime. GitHub masks them in logs as `***`.

### Step A: Get a Docker Hub Access Token

Docker Hub no longer accepts account passwords in pipelines. You must generate an Access Token.

1. Go to [hub.docker.com](https://hub.docker.com) and log in.
2. Navigate to **Account Settings → Settings → Personal Access Token → Genrate new token**.
3. **Access token description = github-actions**
4. Expiration date = None (depends) & Grant it **Read, Write, Delete** permissions.
5. Copy the token immediately (you will not be able to see it again).

### Step B: Inject Secrets into GitHub

You must store your credentials exactly where the runner expects to find them.

1. Go to your GitHub repository.
2. Navigate to **Settings → Secrets and variables → Actions**.
3. Click **New repository secret**.
4. Create `DOCKER_USERNAME` (set value to your Docker Hub username).
5. Create `DOCKER_PASSWORD` (set value to the Access Token you just copied, *not* your actual password).

**How they are referenced in the pipeline:**

```yaml
uses: docker/login-action@v3
with:
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_PASSWORD }}

```

*(Troubleshooting Note: If authentication is denied, ensure the variable names match `${{ secrets.NAME }}` character for character)*.

---

## 4. Tracing Builds with Tags

Images pushed to Docker Hub should never be tagged as `latest` in a CI/CD pipeline. Every build must be traceable back to the exact commit that produced it.

* **The syntax:** `tags: akhiltejadoosari/shopstack-api:${{ github.sha }}`.
* **Short SHA Optimization:** Instead of a massive 40-character SHA, use `${GITHUB_SHA::7}` to grab the first 7 characters (e.g., `cc4fab6`) for cleaner tags in Docker Hub while maintaining uniqueness.
