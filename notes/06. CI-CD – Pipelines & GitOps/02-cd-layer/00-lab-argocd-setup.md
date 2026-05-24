[Home](../README.md) | [GHA Fundamentals](../01-ci-layer/01-actions-fundamentals.md) | [GitOps Bridge](../01-ci-layer/02-production-patterns-gitops-bridge.md) | [ArgoCD Setup](00-lab-argocd-setup.md) | [ArgoCD Ops](01-argocd-operations.md) | [Trivy Scan](../03-security-and-tools/01-trivy-container-scanning.md) | [Interview Prep](../99-interview-prep/README.md)

# Lab — Installing & Exposing ArgoCD

> **Goal:** Install ArgoCD on your k3s cluster, connect it to the ShopStack GitHub repo, and sync the cluster automatically from Git.

---

## 1. Installation (The "Where" & "How")
ArgoCD runs as a set of pods inside your Kubernetes cluster. It is not installed on your laptop; it lives inside the cluster so it can talk to the Kubernetes API server directly.

```bash
# 1. Create a dedicated namespace for ArgoCD
kubectl create namespace argocd

# 2. Install ArgoCD — official manifest from ArgoCD repo
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# 3. Wait for all ArgoCD pods to be Running (takes ~2 minutes)
kubectl get pods -n argocd -w

```

*Wait until you see all pods (server, repo-server, redis, etc.) show `1/1 Running`. Press `Ctrl+C` to exit the watch.*

---

## 2. Exposing the UI

To reach the ArgoCD UI from your browser, we expose it using a `NodePort`.

```bash
# 1. Patch the argocd-server service to NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'

# 2. Find the NodePort assigned
kubectl get svc argocd-server -n argocd

```

*Look for the port mapped to **443** (e.g., `443:30005/TCP`). The `30005` is your port.*

---

## 3. Getting Initial Credentials

ArgoCD generates an admin password automatically on first install.

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

```

*Copy this password. Username is `admin`.*

---

## 4. Troubleshooting (The "Cause & Fix")

If you are stuck, check these three common failure points:

| Symptom | Cause | Fix |
| --- | --- | --- |
| **Browser can't reach UI** | NodePort is blocked by EC2 firewall | AWS Console → EC2 Security Group → Add Inbound Rule → Custom TCP for your NodePort |
| **`kubectl get pods` hangs** | Cluster lacks resources | `kubectl get nodes` — ensure your node is `Ready` |
| **"Invalid metadata.annotations"** | ArgoCD manifest version issue | This is a known harmless warning from the installer; ignore it |
| **UI shows SSL warning** | ArgoCD uses self-signed certs | Click **Advanced → Proceed to...** |

---

## 5. Daily Maintenance Commands

| Action | Command |
| --- | --- |
| **Check ArgoCD Status** | `kubectl get pods -n argocd` |
| **Find UI Port** | `kubectl get svc argocd-server -n argocd` |
| **Restart ArgoCD UI** | `kubectl rollout restart deployment/argocd-server -n argocd` |
