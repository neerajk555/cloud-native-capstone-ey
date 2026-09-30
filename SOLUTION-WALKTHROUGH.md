# Solution Walkthrough

This document explains the reference solution in the `solution/` folder and how to use it - either to
check your own work, or to recover if you get stuck on a specific step.

## How to Use This

**Don't just copy the solution files and run them blindly** - the value of this capstone is in building
each piece yourself and understanding why it works. Use this document to:

1. Check your file content matches, if something isn't working and you suspect a typo
2. Understand WHY each piece is built the way it is (the explanations below go beyond the main
   instructions)
3. Recover a single file if you got stuck, without needing to redo the whole capstone

## Solution File Map

| File | Corresponds to |
|---|---|
| `solution/app/server.js` | Part 1, Step 1.2 |
| `solution/app/Dockerfile` | Part 1, Step 1.3 |
| `solution/helm-chart/values.yaml` | Part 2, Step 2.2 |
| `solution/helm-chart/templates/configmap.yaml` | Part 2, Step 2.3 |
| `solution/helm-chart/templates/deployment.yaml` | Part 2, Step 2.4 |
| `solution/helm-chart/templates/service.yaml` | Part 2, Step 2.5 |
| `solution/argocd/application.yaml` | Part 4, Step 4.4 (you MUST edit the `repoURL` to your own repo - the solution file has a placeholder) |

## Why Each Design Decision Was Made

**Why `eval $(minikube docker-env)` instead of pushing to a real registry?** A real setup (and the AWS
side of this course) pushes images to a registry (ECR, Docker Hub) that Kubernetes then pulls from. This
capstone deliberately avoids that to stay 100% free and offline - minikube's own Docker daemon serving as
a "local registry substitute" is a well-established pattern specifically for local development, not a
shortcut that would work in a real multi-node cluster (a real cluster's nodes don't share a single Docker
daemon the way minikube's one node does).

**Why does the Deployment read `MESSAGE` from a ConfigMap instead of directly from `.Values.message`?**
Either would technically work, but reading it through a ConfigMap and `configMapKeyRef` demonstrates the
REAL pattern used for runtime configuration in Kubernetes - the same mechanism you'd use for a value that
changes without rebuilding the image, matching how this concept was taught earlier in the broader course.

**Why deploy manually with Helm first (Part 2), then hand off to ArgoCD (Part 4), instead of going
straight to ArgoCD?** Isolates failure domains. If Part 2 works but Part 4 doesn't, the problem is in the
GitOps wiring (usually a wrong `repoURL` or path). If Part 2 itself fails, the problem is in the app,
Dockerfile, or chart - and you'd want to know that before adding ArgoCD's extra moving parts on top.

**Why `selfHeal: true`?** This is what separates genuine GitOps from a one-time deployment tool. Without
it, ArgoCD would only push Git's state to the cluster on a change, but would tolerate someone manually
`kubectl edit`-ing something afterward. With it, the cluster continuously and automatically corrects
itself back toward whatever Git currently says - Part 5, Step 5.5 exists specifically to prove this.

## Troubleshooting the Most Common Failure Points

**`ImagePullBackOff` on your Pods** - almost always means either (a) you forgot `eval $(minikube
docker-env)` before running `docker build`, so the image only exists in your system Docker, invisible to
minikube, or (b) `imagePullPolicy` isn't set to `IfNotPresent`, so Kubernetes is trying to pull from a
registry that doesn't have this image. Fix: re-run `eval $(minikube docker-env)`, rebuild the image in
that same terminal, and confirm `imagePullPolicy: IfNotPresent` is in your `deployment.yaml`.

**ArgoCD shows the app as `Unknown` or fails to even find your repo** - almost always a typo in
`repoURL` in `argocd/application.yaml`, or the `path` field not matching your actual folder name. Run
`kubectl describe application capstone-app -n argocd` and read the `Status.Conditions` section - it
usually states the exact problem (e.g., "repository not found" means the URL itself is wrong; "path does
not exist" means the `path` field doesn't match your repo's actual folder structure).

**ArgoCD shows `OutOfSync` and never self-corrects, even with `automated` set** - check that
`syncPolicy.automated` is actually present and correctly indented in your `application.yaml` (a common
YAML indentation mistake makes it look present to a human eye but not apply). You can also force a manual
sync from the UI to confirm the CONNECTION works, even if automation isn't triggering - if a manual sync
succeeds but automatic sync doesn't, the automated policy block itself is the thing to re-check.

**`minikube service capstone-app --url` hangs or shows nothing** - confirm the Service actually exists
(`kubectl get svc`) and that its `type` is `NodePort` (a `ClusterIP` service - the Kubernetes default -
isn't directly reachable this way from outside the cluster).

**Port-forward to the ArgoCD UI stops working after a while** - `kubectl port-forward` is a foreground
process tied to that terminal; if the terminal was closed or the machine slept, just re-run the
port-forward command in a fresh terminal.

**`git push` asks for a password and rejects your normal GitHub password** - GitHub requires a Personal
Access Token instead of your account password for this, as of several years ago now. Create one under
GitHub Settings > Developer Settings > Personal Access Tokens (classic is simplest for this purpose), and
use that token as the password when prompted.

## Final Verification Checklist

- [ ] `docker images` (inside `minikube docker-env`) shows `capstone-app:v1`
- [ ] `helm template ./helm-chart` renders without error and shows `replicas: 2`
- [ ] Your GitHub repo contains `app/`, `helm-chart/`, and `argocd/` with all files committed
- [ ] The ArgoCD UI shows `capstone-app` as **Synced** and **Healthy**
- [ ] Changing `values.yaml`, committing, and pushing results in the live app's response changing,
      without you running `kubectl apply` or `helm upgrade` yourself
- [ ] Manually scaling to 5 replicas gets automatically corrected back to 2 within about a minute
