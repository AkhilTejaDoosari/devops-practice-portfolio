<p align="center">
  <img src="../../assets/cicd-banner.svg" alt="ci-cd" width="100%"/>
</p>

[← devops-journey](../../README.md) | [GHA Fundamentals](./01-ci-layer/01-actions-fundamentals.md) | [GitOps Bridge](./01-ci-layer/02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](./02-cd-layer/00-lab-argocd-setup.md) | [ArgoCD Ops](./02-cd-layer/01-argocd-operations.md) | [Trivy Scan](./03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](./99-interview-prep/README.md)

---

Pipelines, automation, and GitOps — built around ShopStack, the app you containerized in Docker and orchestrated in Kubernetes on AWS EC2.

---

## Why CI/CD — and Why GitHub Actions + ArgoCD

Every `kubectl apply` you ran in Kubernetes was a manual step. You typed it, you watched it, you waited. In a real team that is not sustainable — deployments happen dozens of times a day, from multiple people. One missed step, one wrong image tag, one manual mistake is enough to break production.

CI/CD removes the human from the deployment loop. Code gets pushed, a pipeline runs, an image gets built and tagged, and the cluster updates itself — without anyone typing a single command.

**GitHub Actions** is the CI layer. It is built into the GitHub repo — no separate server, no extra billing. It triggers on events you define, builds the ShopStack images, tags them with the git commit SHA, and pushes them to Docker Hub.

**ArgoCD** is the CD layer. It watches the `infra/k8s/` folder in the ShopStack repo. When a manifest changes — because the CI pipeline updated the image tag — ArgoCD detects the difference between what is in Git and what is running in the cluster, and syncs them. The cluster always reflects what is in Git. That is GitOps.

**Why not Jenkins?** Jenkins requires a dedicated server, ongoing maintenance, and a plugin ecosystem that ages poorly. For a team already on GitHub running Kubernetes, Actions + ArgoCD is the cleanest combination with the least operational overhead.

---

## Prerequisites

**Complete first:** [05. Kubernetes — Orchestration](../05.%20Kubernetes%20–%20Orchestration/README.md)

ArgoCD deploys to a Kubernetes cluster. GitHub Actions builds images that run in Kubernetes. If you do not have ShopStack running on k3s on EC2, CI/CD has nothing to automate.

**What you arrive with:**
- ShopStack full stack running on k3s on AWS EC2
- All six K8s manifests applied and verified (Deployments, Services, ConfigMaps, Secrets, PVC)
- Images already on Docker Hub: `akhiltejadoosari/shopstack-api:1.0`, `shopstack-frontend:1.0`, `shopstack-worker:1.0`
- Both repos on GitHub: `AkhilTejaDoosari/shopstack` (app + manifests) and `AkhilTejaDoosari/devops-journey` (tutorials)

---

## The Running Example — ShopStack

Every file and every lab is built around ShopStack.

| Service | Image | Role |
|---|---|---|
| frontend | `akhiltejadoosari/shopstack-frontend` | Nginx — serves UI, proxies `/api/*` to api |
| api | `akhiltejadoosari/shopstack-api` | FastAPI — business logic, talks to db |
| worker | `akhiltejadoosari/shopstack-worker` | Go — background health pinger |
| db | `postgres:15` | Postgres — all persistent data |

The CI pipeline builds and pushes `api`, `frontend`, and `worker`. The `db` image is upstream — you don't build it.

---

## Where You Take ShopStack

**You arrive:** ShopStack running on k3s. Deployments work. Pods self-heal. Storage persists. Every update requires you to manually build an image, push it, update the manifest, and run `kubectl apply`.

**You leave:** Push code to main → pipeline builds the image → tags it with the commit SHA → pushes it to Docker Hub → updates the manifest in Git → ArgoCD detects the change → cluster syncs automatically. The only manual step left is writing the code.

---

## The Two-Repo Pattern

This module introduces the two-repo pattern you already have set up:

| Repo | What lives here |
|---|---|
| `AkhilTejaDoosari/shopstack` | App source code + `infra/k8s/` manifests + Jenkinsfile + GitHub Actions workflows |
| `AkhilTejaDoosari/devops-journey` | Tutorial files only — what you did, what every file does, how to repeat it |

The CI pipeline lives in the `shopstack` repo and updates `infra/k8s/` manifests when a new image is built. ArgoCD watches `infra/k8s/`. Rolling back is a `git revert` on the manifest. Auditing who deployed what is a `git log`.

---

## Files in This Module

```
06. CI-CD — Pipelines & GitOps/
  ├── README.md                               ← you are here
  ├── 01-ci-layer/
  │   ├── 01-actions-fundamentals.md          ← runner model, workflow anatomy, secrets, SHA tagging
  │   └── 02-production-patterns-gitops-bridge.md  ← path filters, bridge job, race condition fix
  ├── 02-cd-layer/
  │   ├── 00-lab-argocd-setup.md              ← install ArgoCD on k3s, expose UI, get credentials
  │   └── 01-argocd-operations.md             ← Application manifest, sync status, drift, rollbacks
  ├── 03-security-and-tools/
  │   └── 01-trivy-container-scanning.md      ← CVE scanning, push:false, exit-code:1, fixing CVEs
  └── 99-interview-prep/
      └── README.md                           ← 20 questions across GHA + ArgoCD + Trivy
```

---

## Read Order

Read in this order. Each file builds on the previous.

| # | File | What it covers |
|---|---|---|
| 1 | `01-ci-layer/01-actions-fundamentals.md` | What a runner is, workflow file anatomy, secrets, SHA tagging |
| 2 | `01-ci-layer/02-production-patterns-gitops-bridge.md` | Path filters, the bridge job that connects CI to CD, race condition fix |
| 3 | `02-cd-layer/00-lab-argocd-setup.md` | Install ArgoCD on your k3s cluster, expose the UI, get the admin password |
| 4 | `02-cd-layer/01-argocd-operations.md` | Application manifest, Synced vs Healthy, drift, rollback via Git |
| 5 | `03-security-and-tools/01-trivy-container-scanning.md` | Container scanning, CVE severity, how to fix libexpat and Go stdlib CVEs |
| 6 | `99-interview-prep/README.md` | 20 questions — answer out loud, no notes, 30 seconds each |

---

## What You Can Do After This

- Write a GitHub Actions workflow from scratch without documentation
- Build, tag with a commit SHA, and push a Docker image from a CI pipeline
- Explain the difference between CI and CD and why they are separate systems
- Explain why `actions/checkout` is always the first step
- Install and configure ArgoCD on a Kubernetes cluster
- Explain GitOps and why Git is the source of truth for cluster state
- Connect a CI pipeline to CD so deployments happen automatically via the bridge job
- Roll back a deployment by reverting a commit — not with `kubectl rollout undo`
- Debug a failed pipeline run by reading GitHub Actions step logs
- Read a Trivy scan report and fix a CVE by updating a base image

---

## What Comes Next

→ [07. Observability — Monitoring & Logs](../07.%20Observability%20–%20Monitoring%20%26%20Logs/README.md)

CI/CD deploys ShopStack automatically. Observability tells you what is happening inside it after deployment — whether pods are healthy, whether requests are failing, and where to look when they are. ShopStack's API already exposes `/api/metrics` in Prometheus format. That is where the next module begins.
