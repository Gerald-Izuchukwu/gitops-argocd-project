# GitOps with Argo CD — Demo Project

## 1. Project Overview

This project demonstrates a complete **GitOps deployment workflow** using GitHub, Kubernetes (Minikube), and Argo CD. A simple Flask application is containerized, its Kubernetes manifests are stored in Git, and Argo CD is configured to continuously reconcile the live cluster state with the state declared in the Git repository. The project also demonstrates Argo CD detecting and correcting **configuration drift**, and rolling out an application change purely through a Git commit — with no manual `kubectl apply` or `kubectl scale` used as a deployment mechanism.

## 2. Architecture

```
GitHub
   |
   | desired state
   v
Argo CD
   |
   | reconciliation
   v
Minikube
   |
   v
Kubernetes
   |
   v
Application
```

Argo CD continuously polls (or is notified of) the Git repository. Any difference between the manifests in `k8s/` and the live cluster state is detected and reconciled automatically — this is the core GitOps loop.

## 3. Technologies Used

- **Git / GitHub** — source of truth for desired state
- **Docker** — containerizes the Flask application
- **Kubernetes** — orchestrates the application
- **Minikube** — local single-node Kubernetes cluster
- **Argo CD** — GitOps continuous delivery controller
- **WSL** — Linux environment on Windows used to run Minikube/Docker/kubectl
- **Python / Flask** — the demo application

## 4. Project Structure

```
gitops-argocd-project/
├── app/
│   ├── app.py              # Flask application
│   └── requirements.txt    # Python dependencies
├── Dockerfile               # Container build definition
├── k8s/
│   ├── namespace.yaml       # gitops-demo namespace
│   ├── deployment.yaml      # Deployment (replicas, image, labels)
│   └── service.yaml         # Service exposing the app inside/outside Minikube
├── screenshots/             # Evidence images (see section 9)
└── README.md
```

- `app/` — the application source Argo CD ultimately deploys.
- `Dockerfile` — builds the image referenced by `k8s/deployment.yaml`.
- `k8s/` — the **desired state** Argo CD watches and syncs to the cluster. This is the directory registered as the Argo CD Application's `path`.

## 5. Application Deployment

The application was **not** deployed with `kubectl apply`. Instead:

1. The manifests in `k8s/` were pushed to GitHub.
2. An Argo CD `Application` resource was created pointing at this repository and the `k8s/` path, targeting the `gitops-demo` namespace on the in-cluster Minikube destination.
3. Argo CD pulled the manifests from Git and applied them to the cluster on Argo CD's own initiative (automated sync), not via a manual `kubectl` command.
4. All deployment changes afterward (scaling, image/code updates) were made by committing to Git — Argo CD detected and applied every change.

## 6. GitOps Workflow

```
Code/Configuration Change
          ↓
        GitHub
          ↓
       Argo CD
          ↓
     Detect Change
          ↓
     Reconciliation
          ↓
      Kubernetes
          ↓
       Application
```

Every change in this project — replica count, application version — followed this same loop: a change was committed and pushed to GitHub, Argo CD's controller detected the diff between Git and the live cluster, and Argo CD reconciled the cluster to match Git, with Kubernetes then rolling out the new state to the running Pods.

## 7. Configuration Drift

**Experiment:** After Git declared `replicas: 3`, the live Deployment was manually scaled with:

```bash
kubectl scale deployment gitops-demo --replicas=1 -n gitops-demo
```

- **Desired state (Git):** `replicas: 3`
- **Actual state (cluster, right after the manual scale):** `replicas: 1`
- **Argo CD status:** The Application showed `OutOfSync` — Argo CD detected that the live state no longer matched Git.
- **Reconciliation:** Because automated sync (with self-heal) was enabled, Argo CD reverted the manual change and scaled the Deployment back to match Git.
- **Final state:** `replicas: 3` — Git's declared state won, demonstrating that Kubernetes is not the source of truth; Git is.

## 8. Application Update

The application was updated from:

> GitOps deployment with Argo CD is working!

to:

> Version 2 deployed using GitOps and Argo CD!

by editing `app/app.py`, rebuilding/pushing the container image (with a new tag), updating the image tag in `k8s/deployment.yaml`, committing, and pushing. Argo CD detected the manifest change and rolled out new Pods running the updated image — no manual `kubectl set image` or `kubectl apply` was used.

## 9. Screenshots

Evidence stored in `screenshots/`:

| File | Evidence of |
|---|---|
| `minikube.png` | Minikube cluster running (`minikube status`, `kubectl get nodes`) |
| `argocd-dashboard.png` | Argo CD web UI logged in |
| `argocd-application.png` | The `gitops-demo` Argo CD Application view |
| `synced.png` | Application in `Synced` / `Healthy` state |
| `out-of-sync.png` | Application in `OutOfSync` state during the drift experiment |
| `drift-recovery.png` | Application back to `Synced` after Argo CD self-heal |
| `application.png` | The running application response in a browser |

### Sync Status vs Health Status

- **Sync Status** — whether the *live cluster state* matches the *desired state in Git*.
  - `Synced` — cluster matches Git exactly.
  - `OutOfSync` — cluster has drifted from Git (e.g. after a manual `kubectl scale`).
- **Health Status** — whether the *deployed resources themselves* are working correctly, independent of Git.
  - `Healthy` — Pods/Deployment are running as expected.
  - `Progressing` — a rollout is in progress (e.g. new Pods starting after a sync).
  - `Degraded` — resources exist but are failing (e.g. CrashLoopBackOff).

A deployment can be `Synced` but `Degraded` (matches Git, but the Pods are crashing), or `OutOfSync` but still `Healthy` (drifted from Git, but the currently-running Pods are fine) — these two axes are independent.
