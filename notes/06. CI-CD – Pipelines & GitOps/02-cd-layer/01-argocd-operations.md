[Home](../README.md) | [GHA Fundamentals](../01-ci-layer/01-actions-fundamentals.md) | [GitOps Bridge](../01-ci-layer/02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](00-lab-argocd-setup.md) | [ArgoCD Ops](01-argocd-operations.md) | [Trivy Scan](../03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](../99-interview-prep/README.md)

# ArgoCD — GitOps Operations & Architecture

> **The GitOps Concept:** GitHub Actions built the image and pushed it to Docker Hub, but nothing deployed it. ArgoCD eliminates the need to run `kubectl apply` manually. It watches your GitHub repository, and when a manifest changes, it syncs the cluster automatically. **Git is the single source of truth. You never run `kubectl apply` in production again**.

---

## Table of Contents
- [1. The Full Application Template (With Annotations)](#1-the-full-application-template-with-annotations)
- [2. Understanding Sync & Health Statuses](#2-understanding-sync--health-statuses)
- [3. Configuration Drift & Self-Healing](#3-configuration-drift--self-healing)
- [4. Rollbacks: The GitOps Way](#4-rollbacks-the-gitops-way)
- [5. Troubleshooting & Daily Operations](#5-troubleshooting--daily-operations)

---

## 1. The Full Application Template (With Annotations)

An Application is the Custom Resource Definition (CRD) that tells ArgoCD what Git repository to watch, which specific folder to look at, and which cluster to deploy to.

```yaml
# ---------------------------------------------------------
# ARGOCD APPLICATION DEFINITION
# ---------------------------------------------------------
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shopstack                          # The name of the app inside the ArgoCD UI
  namespace: argocd                        # Where ArgoCD itself is installed
spec:
  project: default
  
  # ---------------------------------------------------------
  # SOURCE: Where is the code coming from?
  # ---------------------------------------------------------
  source:
    repoURL: [https://github.com/AkhilTejaDoosari/shopstack](https://github.com/AkhilTejaDoosari/shopstack)  # The exact Git repository to watch
    targetRevision: HEAD                                    # Always watch the latest commit on the default branch
    path: infra/k8s                                         # Watch ONLY this folder (ignores changes to src/ or READMEs)
  
  # ---------------------------------------------------------
  # DESTINATION: Where is the code going?
  # ---------------------------------------------------------
  destination:
    server: [https://kubernetes.default.svc](https://kubernetes.default.svc)                  # Deploy to the same cluster ArgoCD is running on
    namespace: default                                      # Deploy the application pods into the 'default' namespace
  
  # ---------------------------------------------------------
  # SYNC POLICY: How should ArgoCD behave when Git changes?
  # ---------------------------------------------------------
  syncPolicy:
    automated:                                              # Polls Git every 3 minutes and applies changes automatically
      prune: true                                           # If a manifest is deleted from Git, delete it from the cluster
      selfHeal: true                                        # If someone edits the cluster manually, immediately revert it to match Git

```

---

## 2. Understanding Sync & Health Statuses

As an operator, reading the ArgoCD UI statuses is critical.    
They are split into two categories:    
**1. Sync** (Does the config match?)    
**2. Health** (Is the application actually running?).   

### SYNC STATUS (Config State)

* **Synced:** The cluster configuration matches Git exactly. This is the normal, healthy state.
* **OutOfSync:** The cluster differs from Git. This happens during the 3-minute polling window before a sync, OR when someone bypasses GitOps and manually runs `kubectl` commands directly against the cluster.

### HEALTH STATUS (Runtime State)

* **Healthy:** All pods are running, services are routing correctly, and replica limits are satisfied.
* **Degraded:** Something is actively failing at runtime (e.g., a pod is in `CrashLoopBackOff`, an image pull failed, or a PVC is unbound).
* **Progressing:** An update is currently rolling out. Old pods are terminating, and new pods are starting.
* **Missing:** A resource is defined in Git but is completely missing from the cluster (usually a pathing error or a blocked apply).

---

## 3. Configuration Drift & Self-Healing

**What is Drift?**
If an engineer tries to bypass the pipeline and manually scales a deployment using the terminal (`kubectl scale deployment shopstack-api --replicas=5`), they create "Drift". Git says there should be 2 replicas, but the cluster now has 5.

**How ArgoCD Handles It:**
Because `selfHeal: true` is defined in the Application template, ArgoCD will immediately flag the status as **OutOfSync** and instantly kill the 3 extra pods. It actively forces the cluster to revert back to the exact state defined in Git (2 replicas).

---

## 4. Rollbacks: The GitOps Way

If a bad image is deployed and production breaks, **DO NOT** use `kubectl rollout undo`. Rolling back via `kubectl` breaks the GitOps contract, leaves no audit trail, and causes ArgoCD to fight your manual changes.

**Always execute rollbacks via Git history:**

```bash
# 1. Find the manifest commit that broke the environment (e.g., "ci: update api image")
git log --oneline -10

# 2. Revert that specific commit (this creates a new commit restoring the old image tag)
git revert COMMIT_SHA --no-edit

# 3. Push the revert back to GitHub
git push origin main

```

ArgoCD detects the new revert commit, sees the old image tag is back in the manifest, and automatically syncs the cluster back to the stable state.

---

## 5. Troubleshooting & Daily Operations

| Symptom | Cause & Fix |
| :--- | :--- |
| **Status is Synced + Degraded** | **Cause:** The K8s manifest was applied perfectly from Git (`Synced`), but the container itself is crashing inside the cluster (`Degraded`).<br><br>**Fix:** Run `kubectl describe pod` to find the crash reason (e.g., bad code, missing env vars). Fix the code and push a new image. |
| **ArgoCD shows OutOfSync** | **Cause:** Someone edited the cluster manually, or a sync hasn't polled yet.<br><br>**Fix:** Wait 3 minutes, or click "Sync" in the UI to force an immediate poll. |
| **App shows "Missing"** | **Cause:** The `path` defined in your Application (`infra/k8s`) cannot be found in the repo.<br><br>**Fix:** Confirm the folder path matches the Git repository exactly. |
| **New image not deploying** | **Cause:** The GitHub Actions pipeline pushed to Docker Hub, but the `update-manifest` bridge job failed.<br><br>**Fix:** Check your CI pipeline to ensure the manifest file was successfully updated with the new tag. |
