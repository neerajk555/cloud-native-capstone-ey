# Capstone: Local GitOps Pipeline with Docker, Kubernetes, Helm, and ArgoCD

## Goal

Take a small application from source code to a running, self-healing Kubernetes deployment that
**deploys itself automatically whenever you push a change to Git** - entirely on your own machine, no
cloud account, no cost, no external dependencies beyond a free GitHub repository.

By the end of this capstone you will have built the same fundamental pipeline real companies use to
ship software to Kubernetes - just running locally end to end instead of against a cloud cluster.

## What You'll Build

```
Your Code  --->  Docker Image  --->  Helm Chart  --->  Git Repo (GitHub)
                                                              |
                                                              v
                                                      ArgoCD (watching your repo)
                                                              |
                                                              v
                                                   Local Kubernetes (minikube)
```

1. A small Node.js web app, containerized with Docker
2. A Helm chart packaging that app's Deployment, Service, and ConfigMap
3. A GitHub repository holding that Helm chart
4. ArgoCD, running inside your local minikube cluster, watching that repository
5. Proof that changing a value, committing, and pushing results in an **automatic** deployment - no
   `kubectl apply`, no `helm upgrade` typed by hand

## Time Budget (3 hours total)

| Part | What | Time |
|---|---|---|
| 0 | Environment check | 15 min |
| 1 | Build and load the container image | 30 min |
| 2 | Package and deploy with Helm (manually first) | 40 min |
| 3 | Push the chart to GitHub | 20 min |
| 4 | Install ArgoCD and connect it to your repo | 35 min |
| 5 | Prove GitOps works - change, push, watch it deploy | 30 min |
| - | Buffer / troubleshooting | 10 min |

## Prerequisites

- Ubuntu with Docker, `kubectl`, `minikube`, and `helm` already installed
- VS Code installed
- A free [GitHub](https://github.com) account
- Git configured locally (`git config --global user.name "..."` and `user.email`)

## What This Capstone Deliberately Does NOT Use

- **No AWS, no cloud account of any kind** - everything runs on your own machine
- **No agentic AI / LLMs** - this is pure infrastructure and deployment automation
- **No real container registry** - the image is built directly inside minikube's own Docker daemon, so
  there's nothing to push anywhere or pay for

## Files in This Repository

```
00-README.md                    This file
CAPSTONE-INSTRUCTIONS.md        The step-by-step instructions - start here
SOLUTION-WALKTHROUGH.md         Reference solution + how to verify your work against it
solution/
  app/
    server.js                   The sample app
    Dockerfile
  helm-chart/
    Chart.yaml
    values.yaml
    templates/
      deployment.yaml
      service.yaml
      configmap.yaml
  argocd/
    application.yaml            The ArgoCD Application definition
```

Start with `CAPSTONE-INSTRUCTIONS.md`.
