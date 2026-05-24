[Home](../README.md) | [GHA Fundamentals](01-actions-fundamentals.md) | [GitOps Bridge](02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](../02-cd-layer/00-lab-argocd-setup.md) | [ArgoCD Ops](../02-cd-layer/01-argocd-operations.md) | [Trivy Scan](../03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](../99-interview-prep/README.md)

# GitHub Actions — Production Patterns & The GitOps Bridge

> **The Bridge Concept:** If GitHub Actions builds the image and pushes it to Docker Hub, but no one updates the Kubernetes manifests, ArgoCD will never deploy the new code. **The pipeline must update the `deployment.yaml` file with the new image SHA and push that change back to GitHub.** This manifest update is "The Bridge" between CI and CD.

---

## Table of Contents
- [1. The Full Workflow Template (With Annotations)](#1-the-full-workflow-template-with-annotations)
- [2. Pipeline Isolation: The paths: Filter](#2-pipeline-isolation-the-paths-filter)
- [3. The update-manifest Job (The Bridge)](#3-the-update-manifest-job-the-bridge)
- [4. Solving Race Conditions: Stash, Rebase, Pop](#4-solving-race-conditions-stash-rebase-pop)

---

## 1. The Full Workflow Template (With Annotations)

This is the standard pipeline structure utilizing all production patterns: path filters, short SHA tags, and the GitOps bridge with race-condition handling.

```yaml
# ---------------------------------------------------------
# 1. THE TRIGGER (PIPELINE ISOLATION)
# ---------------------------------------------------------
on:
  push:
    branches: [main]
    paths:
      - 'services/frontend/**'                 # Only trigger if frontend code changes
      - '.github/workflows/build-frontend.yml' # Or if this pipeline file itself changes

name: Build and Push Frontend

jobs:
# ---------------------------------------------------------
# 2. THE CI JOB (BUILD & PUSH)
# ---------------------------------------------------------
  build-and-push:
    runs-on: ubuntu-latest                     # Spins up a brand new, disposable Ubuntu VM specifically for this job
    outputs:
      image_tag: ${{ steps.tag.outputs.sha }}  # Export the SHA so the next job can use it

    steps:
      - name: Checkout code
        uses: actions/checkout@v4              # Clones repo onto the blank runner

      - name: Set short SHA tag
        id: tag
        run: echo "sha=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT # Grabs first 7 chars of commit hash

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }} # Injected securely from GitHub Settings

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: ./services/frontend
          push: true
          tags: akhiltejadoosari/shopstack-frontend:${{ steps.tag.outputs.sha }}
          
      # ---> END OF JOB: As soon as this step finishes, this specific ubuntu-latest VM 
      # ---> is permanently destroyed. All cloned files and memory are wiped clean.

# ---------------------------------------------------------
# 3. THE CD BRIDGE JOB (UPDATE MANIFEST)
# ---------------------------------------------------------
  update-manifest:
    runs-on: ubuntu-latest                     # Spins up a completely NEW, blank VM for this job
    needs: build-and-push                      # Strict dependency: wait for build to succeed

    steps:
      - name: Checkout code
        uses: actions/checkout@v4              # Must clone repo again because this is a new VM

      - name: Update manifest
        run: |
          # sed -i edits the file in-place. It finds the old tag and replaces it with the new SHA.
          sed -i "s|akhiltejadoosari/shopstack-frontend:.*|akhiltejadoosari/shopstack-frontend:${{ needs.build-and-push.outputs.image_tag }}|" \
            infra/k8s/frontend-deployment.yaml

      - name: Commit and push manifest
        run: |
          # Configure Git bot identity
          git config user.name "Your-Name"
          git config user.email "you@example.com"
          
          # RACE CONDITION FIX: Handle simultaneous pipeline pushes safely
          git add -A
          git stash                            # 1. Hide the sed changes on a temporary shelf
          git pull --rebase                    # 2. Fetch commits that beat you to the repo
          git stash pop                        # 3. Drop your hidden changes back on top
          
          # Stage, commit, and push back to GitHub
          git add infra/k8s/frontend-deployment.yaml
          git commit -m "ci: update frontend image to ${{ needs.build-and-push.outputs.image_tag }}" || echo "No changes"
          git push
          
      # ---> END OF JOB: The second VM is now permanently destroyed.

```

> **Security Note:** For the `update-manifest` job to push back to the repository, the GitHub Actions bot requires Write permissions. This must be enabled manually in the repository via `Settings → Actions → General → Workflow permissions → Read and write permissions`.

---

## 2. Pipeline Isolation: The `paths:` Filter

In a repository with multiple services (e.g., API, frontend, worker), a single code push should not trigger every single pipeline.

Without `paths:`, every push triggers all three pipelines — even if only one service changed. Path filters ensure only the relevant pipeline runs, saving build minutes and providing faster feedback.

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'                      # Only run if API code changes
      - '.github/workflows/build-api.yml'      # Or if this pipeline file changes

```

---

## 3. The `update-manifest` Job (The Bridge)

Understanding exactly what the bridge block is doing under the hood:

### The Dependency

```yaml
    needs: build-and-push

```

Tells GitHub: *"Do not start this job until the image has been successfully built and pushed."* If the build fails, the manifest update is skipped.

### The `sed` Command

```yaml
          sed -i "s|akhiltejadoosari/shopstack-frontend:.*|akhiltejadoosari/shopstack-frontend:${{ needs.build-and-push.outputs.image_tag }}|" \

```

`sed` is a Linux stream editor used to find and replace text.

* **`sed -i`**: Edits the file "in-place" (saves changes directly).
* **`akhiltejadoosari/shopstack-frontend:.*`**: The pattern it looks for. The `.*` means "match everything after the colon." It doesn't matter what the old Git SHA was.
* **`${{ needs.build-and-push.outputs... }}`**: Replaces the old tag with the brand new short SHA generated in the previous job.

---

## 4. Solving Race Conditions: Stash, Rebase, Pop

**The Problem:** When multiple pipelines run simultaneously (e.g., frontend and worker) and race to push their manifest updates back to the repository, the second pipeline will get a `! [rejected] (fetch first)` error because the remote branch moved ahead of its local runner state.

**The Fix:** You must pull the latest changes and rebase your runner's local commits before pushing. However, because the runner used `sed` to edit the manifest file, `git pull --rebase` will refuse to run with unstaged changes.

**The Correct Order of Operations:**

```bash
git add -A
git stash             # Saves the sed manifest changes to the side
git pull --rebase     # Updates the branch with commits from other pipelines
git stash pop         # Reapplies the sed changes on top
git commit -m "ci: update image tag" || echo "No changes"
git push

```

*(Note: `|| echo "No changes"` is required so the job doesn't fail and exit with code 1 if there was nothing new to commit).*
