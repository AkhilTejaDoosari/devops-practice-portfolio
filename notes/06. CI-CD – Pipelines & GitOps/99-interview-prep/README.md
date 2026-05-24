[Home](../README.md) | [GHA Fundamentals](../01-ci-layer/01-actions-fundamentals.md) | [GitOps Bridge](../01-ci-layer/02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](../02-cd-layer/00-lab-argocd-setup.md) | [ArgoCD Ops](../02-cd-layer/01-argocd-operations.md) | [Trivy Scan](../03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](README.md)

## CI/CD Interview Prep

**Goal:** Answer every question below out loud, no notes, in under 30 seconds each. If you can't — reread the relevant file, then come back.

---

## GitHub Actions

---

**1. What is a GitHub Actions runner?**

A runner is a disposable Linux VM that GitHub spins up on demand. It starts completely blank — no files, no credentials, no packages beyond base Ubuntu. It runs your steps top to bottom and is destroyed permanently when the job finishes. This is why `actions/checkout` is always the first step — without it, the runner has zero files to work with. The blank slate is a feature, not a limitation. Every run is a guaranteed clean environment.

---

**2. Why does every workflow start with `actions/checkout`?**

The runner is a blank machine. It has no knowledge of your repository. `actions/checkout` clones your GitHub repo onto the runner. Without it, the very next step — whether it's a Docker build, a test, or a linter — would fail immediately because there are no files to operate on. It's not optional and it's not automatic. You declare it explicitly because on a blank machine, nothing is assumed.

---

**3. Where do secrets live and how do you reference them in a workflow?**

Secrets live in GitHub repository Settings → Secrets and variables → Actions. You never put credentials in the workflow YAML — the file is version controlled and visible to anyone with repo access. Inside the workflow you reference a secret as `${{ secrets.SECRET_NAME }}`. GitHub injects the value at runtime as an environment variable and masks it in logs as `***`. For ShopStack, `DOCKER_USERNAME` and `DOCKER_PASSWORD` are stored there and referenced in the `docker/login-action` step.

---

**4. What is the difference between a job and a step?**

A step is a single action — one command, one shell script, one call to a marketplace action. Steps run sequentially on the same runner. A job is a group of steps. Jobs run in parallel by default. If you need one job to wait for another — for example, `update-manifest` must wait for `build-and-push` to succeed — you declare `needs: build-and-push`. Each job gets its own fresh runner, which is why you must `actions/checkout` again at the start of the second job.

---

**5. What does `paths:` do in the trigger block and why does it matter in a monorepo?**

`paths:` filters which pushes actually trigger the workflow. Without it, a push to any file in the repository triggers every pipeline — pushing a README change triggers the API build, the frontend build, and the worker build. In ShopStack, `paths: - 'services/api/**'` means the API pipeline only runs when API source files change. This saves build minutes and gives faster, more relevant feedback. Each service has its own workflow with its own path filter.

---

**6. Why tag images with a short SHA instead of `latest`?**

`latest` is meaningless for tracing. If a deployment breaks, you cannot tell which code produced the `latest` image. A short SHA like `cc4fab6` maps directly to an exact commit in Git — you can `git show cc4fab6` and see every line that changed. In ShopStack, the short SHA is generated with `${GITHUB_SHA::7}` and used as the image tag. Every image is traceable. Every deployment is auditable.

---

**7. What is the GitOps bridge and why does it exist?**

GitHub Actions builds the image and pushes it to Docker Hub. But ArgoCD watches the Git repo — not Docker Hub. If the manifest still shows the old image tag, ArgoCD will never deploy the new image. The bridge is the `update-manifest` job that runs after a successful build. It uses `sed` to replace the old image tag in the deployment YAML with the new SHA, commits the change, and pushes it back to GitHub. ArgoCD sees the manifest change and syncs the cluster. Without the bridge, CI and CD are disconnected.

---

**8. What is the race condition in the update-manifest job and how do you fix it?**

When two pipelines run simultaneously — say frontend and worker both push at the same time — both try to commit a manifest change back to the same branch. The second one gets `rejected: fetch first` because the first already moved the remote branch ahead. The fix is stash → rebase → pop: `git stash` hides the `sed` changes, `git pull --rebase` pulls the latest commits from the first pipeline, `git stash pop` reapplies your changes on top. Then you commit and push cleanly on top of the updated branch.

---

## ArgoCD

---

**9. What is GitOps and what problem does ArgoCD solve?**

Before GitOps, deploying meant someone running `kubectl apply` manually — on their laptop, with their own credentials, with no record of what was deployed or when. GitOps makes Git the single source of truth. The cluster must look exactly like what is in the repo — no more, no less. ArgoCD enforces this. It watches your Git repo, and when a manifest changes, it syncs the cluster automatically. No manual `kubectl apply` in production. No drift. Full audit trail in Git history.

---

**10. What is an ArgoCD Application manifest?**

It's a Kubernetes custom resource that tells ArgoCD three things: what Git repo to watch, which folder inside that repo to look at, and which cluster and namespace to deploy to. For ShopStack it points to `github.com/AkhilTejaDoosari/shopstack`, watches `infra/k8s/`, and deploys to the `default` namespace on the local cluster. The `syncPolicy.automated` block with `prune: true` and `selfHeal: true` makes it fully hands-off — ArgoCD polls every 3 minutes and applies any changes automatically.

---

**11. What is the difference between Synced and Healthy?**

They measure two different things. Synced means the cluster configuration matches Git exactly — the manifests that are in Git are what's applied to the cluster. Healthy means the application is actually running correctly — pods are up, services are routing, replicas are satisfied. A deployment can be Synced but Degraded: ArgoCD applied the manifest perfectly, but the container is crashing inside the cluster. That combination — Synced + Degraded — means the bug is in the code or config, not in the manifest.

---

**12. What is configuration drift and how does ArgoCD handle it?**

Drift is when the cluster state no longer matches Git. The most common cause: an engineer bypasses the pipeline and runs `kubectl scale deployment shopstack-api --replicas=5` directly. Git says 2 replicas. The cluster now has 5. ArgoCD detects this immediately because it continuously compares the live cluster state against Git. With `selfHeal: true`, it doesn't wait — it kills the 3 extra pods and restores the cluster to exactly what Git defines.

---

**13. How do you roll back a bad deployment in ArgoCD?**

You revert in Git — not with `kubectl rollout undo`. Using `kubectl` directly breaks the GitOps contract, creates drift, and leaves no audit trail. ArgoCD would detect the manual rollback and immediately try to fight it back to whatever Git says. The correct way: `git log --oneline` to find the bad commit, `git revert COMMIT_SHA --no-edit` to create a new commit that restores the old image tag, `git push`. ArgoCD sees the revert commit, detects the old tag in the manifest, and syncs the cluster back to the stable image.

---

**14. What does OutOfSync mean and what causes it?**

OutOfSync means the live cluster differs from what Git defines. Two causes: first, someone made a manual change with `kubectl` directly — drift. Second, ArgoCD polled Git, found a manifest change, but hasn't applied it yet — it polls every 3 minutes, so there's a short window. In ShopStack with `automated` sync, OutOfSync is usually resolved automatically within 3 minutes. If it persists, check whether the sync is failing — `kubectl get pods -n argocd` to verify ArgoCD itself is healthy.

---

## Trivy

---

**15. What is Trivy and why is it in the pipeline?**

Trivy is a container image vulnerability scanner. Every Docker image inherits hundreds of OS packages from its base image — packages you didn't write and didn't choose. Any of those packages could have a known CVE. Without scanning, you push the image, ArgoCD deploys it, and it runs in production while attackers read the same public CVE database to find exactly what version of libexpat or OpenSSL is running in your cluster. Trivy sits between the build and the push — it scans every package and blocks the push if it finds a critical exploitable vulnerability.

---

**16. Why do you build with `push: false` and `load: true` before scanning?**

If you push first and scan second, the vulnerable image is already in Docker Hub before results come back. ArgoCD could deploy it in the 3 minutes before the scan completes. `push: false` keeps the image off Docker Hub entirely — it stays in the runner's local Docker daemon. `load: true` makes the image visible to Trivy in that local daemon. Trivy scans it locally. Only if the scan passes does the actual push step run. The image never reaches the registry with a known critical CVE.

---

**17. What happens if you forget `exit-code: 1` in the Trivy step?**

Trivy still runs, downloads the CVE database, scans the image, and prints the vulnerability table in the Actions log. But it exits with code 0 regardless of what it finds. GitHub Actions only fails a step on non-zero exit codes. Without `exit-code: 1`, the pipeline passes, the push step runs, and the vulnerable image reaches Docker Hub. The scan becomes a reporting tool with no enforcement power. Adding `exit-code: 1` is what makes Trivy a gate, not just a log.

---

**18. Why block on `CRITICAL` only and not `HIGH`?**

A typical Alpine base image has dozens of HIGH vulnerabilities. Most have no fix yet. Most require specific conditions to exploit that don't apply to your environment. Blocking on HIGH means the pipeline almost never passes and engineers start ignoring the tool. CRITICAL is the threshold where active exploits exist in the wild — real attackers using the vulnerability right now against real targets. That's worth blocking a deployment for. HIGH gets logged but doesn't stop the pipeline.

---

**19. What does `ignore-unfixed: true` do?**

It tells Trivy to skip CVEs where no patch has been released yet. If a CVE exists but the package maintainers haven't fixed it, there is nothing you can do — you cannot patch what has no patch. Blocking the pipeline on an unfixable CVE stops all deployments indefinitely for zero security benefit. When a fix is eventually released, Trivy catches it automatically on the next scan. `ignore-unfixed: true` keeps the pipeline focused on what's actually actionable.

---

**20. A Trivy scan found a CVE in `stdlib` for the Go worker — not in an Alpine package. How is that different?**

Alpine package CVEs are fixed by upgrading the base OS image — the patched package ships in the newer Alpine version. But `stdlib` is the Go standard library compiled directly into the binary during `go build`. It's not an OS package you can `apk upgrade`. The CVE is in the Go compiler itself. The fix is to upgrade the builder stage in the multi-stage Dockerfile — `FROM golang:1.22-alpine AS builder` to `FROM golang:1.24-alpine AS builder`. The binary gets recompiled with the patched Go runtime and the CVE is no longer present in the binary.

---

## The Questions That Catch People

These are the traps interviewers use to separate people who followed a tutorial from people who understand the system.

---

**"GitHub Actions or ArgoCD — which one deploys to the cluster?"**

ArgoCD. GitHub Actions builds the image and pushes it to Docker Hub. It never touches the cluster. ArgoCD watches Git for manifest changes and syncs the cluster. They are two separate systems. The connection between them is a manifest file — CI updates the image tag, ArgoCD sees the change and deploys.

---

**"You pushed a fix but the cluster didn't update. Where do you look first?"**

Three places in order. First: did the GitHub Actions pipeline succeed? Check the Actions tab — maybe the build failed, maybe the push failed, maybe the `update-manifest` job failed. Second: did the manifest actually change? Check `git log` on `infra/k8s/` — if the image tag wasn't updated, ArgoCD has nothing to sync. Third: is ArgoCD healthy? `kubectl get pods -n argocd` — if the ArgoCD server is down, no syncs happen.

---

**"Trivy passed but you later found a vulnerability in production. How?"**

Three possible reasons. One: `ignore-unfixed: true` skipped a CVE that had no fix at scan time, and a fix was released after the build. The package is still vulnerable until you rebuild with the patched base image. Two: the CVE was below CRITICAL threshold — HIGH or MEDIUM — and was not blocked. Three: the vulnerability is not in an OS package or language runtime — it's in application code itself, which Trivy doesn't scan. Trivy covers the image's package dependencies, not your business logic.

---

## Checkpoint

Before closing this file — answer these four out loud:

1. A developer pushes to main. Walk me through what happens from push to running pod.
2. What's the difference between Synced+Degraded and OutOfSync+Healthy?
3. Why can't you just `kubectl rollout undo` when a deployment goes bad?
4. Your pipeline is passing Trivy but pushing vulnerable images. What's misconfigured?

If all four are clean — you're ready.
