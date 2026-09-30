# Capstone Instructions

Follow these parts in order. Each part states its time budget - if you're running long on one part,
move on and come back if you have buffer time at the end; don't let one stuck step eat the whole session.

---

## Part 0: Environment Check (15 min)

Confirm every tool is installed and working before writing any code - catching a missing tool now is far
less frustrating than discovering it mid-exercise.

```
docker --version
minikube version
kubectl version --client
helm version
git --version
```

Every command should print a real version number. If any command isn't found, install it before
continuing - this capstone assumes all five are already working.

**Start minikube** with enough resources for ArgoCD to run alongside your app:

```
minikube start --driver=docker --cpus=4 --memory=8192
```

This can take a few minutes on first run. Confirm it's actually up:

```
kubectl get nodes
```

You should see one node in `Ready` status.

---

## Part 1: Build and Load the Container Image (30 min)

### Step 1.1: Create your project folder

**In VS Code:** File > Open Folder, create a new folder called `capstone-local-gitops`, and open it.
Inside, create a subfolder `app`.

### Step 1.2: Write the app

**In VS Code:** create `app/server.js`:

```javascript
const http = require('http');

const VERSION = process.env.VERSION || 'v1';
const MESSAGE = process.env.MESSAGE || 'default message - no ConfigMap applied';

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('healthy');
    return;
  }
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({
    version: VERSION,
    message: MESSAGE,
    hostname: require('os').hostname(),
  }, null, 2));
});

server.listen(3000, () => console.log(`Listening on port 3000, version ${VERSION}`));
```

Notice `hostname` is included in the response - this will matter later when you scale to multiple
replicas and want to see requests actually landing on different Pods.

### Step 1.3: Write the Dockerfile

**In VS Code:** create `app/Dockerfile`:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

No `npm install` needed - this app deliberately uses only Node's built-in `http` module, so the build
stays fast and there's nothing to go wrong with a missing `package.json`.

### Step 1.4: Point your terminal's Docker at minikube's internal Docker daemon

```
eval $(minikube docker-env)
```

**This is the single most important command in this whole capstone to understand.** minikube runs its
own internal Docker daemon, separate from your regular system Docker. By default, `docker build` talks
to your SYSTEM Docker - an image built there is invisible to Kubernetes running inside minikube. This
command temporarily redirects your terminal's `docker` command to build directly inside minikube's own
Docker daemon instead, so whatever you build next is immediately usable by Kubernetes with no separate
push or registry step.

**This only affects the current terminal window** - if you open a new terminal later, you'll need to run
this command again before building anything else, or Kubernetes won't be able to find your image.

### Step 1.5: Build the image

```
cd app
docker build -t capstone-app:v1 .
```

### Step 1.6: Confirm minikube can see it

```
docker images | grep capstone-app
```

You should see `capstone-app` with tag `v1` in the list - proof the image landed inside minikube's
Docker daemon, not just your system's.

### Verify Part 1

```
docker run --rm -p 3000:3000 -e MESSAGE="test message" capstone-app:v1 &
sleep 2
curl http://localhost:3000/
curl http://localhost:3000/health
kill %1
```

You should see a JSON response with `"message": "test message"` and a plain `healthy` response - confirm
the app works correctly BEFORE putting Kubernetes and Helm in the mix, so any problem you hit later is
clearly a Kubernetes/Helm issue, not an app bug.

---

## Part 2: Package and Deploy with Helm - Manually First (40 min)

You'll deploy this manually with Helm once, confirm it works, THEN hand control over to ArgoCD in Part
4. Deliberately doing this in two stages means that if something breaks later, you'll already know
whether the app/chart itself is the problem (caught here) or the GitOps wiring is (caught in Part 4).

### Step 2.1: Scaffold the chart

```
cd ..
helm create helm-chart
```

This generates a full working chart with a lot of content you don't need for this capstone. Delete the
extra template files it creates by default, keeping only the folder structure:

```
rm helm-chart/templates/*.yaml
rm -rf helm-chart/templates/tests
rm helm-chart/values.yaml
```

### Step 2.2: Write `values.yaml`

**In VS Code:** create `helm-chart/values.yaml`:

```yaml
replicaCount: 2

image:
  repository: capstone-app
  tag: v1
  pullPolicy: IfNotPresent

message: "Hello from the capstone - v1"

service:
  type: NodePort
  port: 3000
```

`pullPolicy: IfNotPresent` matters here specifically because of Part 1's `minikube docker-env` step -
this tells Kubernetes "use the image if it's already there, don't try to pull it from a registry" -
which is exactly right, since there IS no registry in this setup; the image only exists inside
minikube's own Docker daemon.

### Step 2.3: Write the ConfigMap template

**In VS Code:** create `helm-chart/templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  MESSAGE: {{ .Values.message | quote }}
```

### Step 2.4: Write the Deployment template

**In VS Code:** create `helm-chart/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 3000
          env:
            - name: VERSION
              value: {{ .Values.image.tag | quote }}
            - name: MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: {{ .Release.Name }}-config
                  key: MESSAGE
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 10
```

Notice `MESSAGE` comes from the ConfigMap (via `configMapKeyRef`), not directly from `.Values.message` -
this is deliberate: it proves the ConfigMap is actually wired in correctly, rather than the app just
reading a value that happened to be passed some other way.

### Step 2.5: Write the Service template

**In VS Code:** create `helm-chart/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 3000
```

### Step 2.6: Render the chart locally before deploying - catch mistakes early

```
helm template capstone-app ./helm-chart
```

This prints the fully-rendered YAML to your terminal without touching the cluster at all - read through
it and confirm `replicas: 2`, the image name, and the ConfigMap reference all look right. Catching a
templating mistake here is much faster than debugging a failed deployment.

### Step 2.7: Deploy it manually with Helm

```
helm install capstone-app ./helm-chart
```

### Step 2.8: Verify it's actually running

```
kubectl get pods
kubectl get svc
```

Wait until both Pods show `Running` and `READY 1/1` (the readiness probe needs to pass first).

### Step 2.9: Reach the app

```
minikube service capstone-app --url
```

This prints a URL - `curl` it:

```
curl $(minikube service capstone-app --url)/
```

### Verify Part 2

- `kubectl get pods` shows 2 Pods, both `Running` and `1/1` ready
- The `curl` response shows `"message": "Hello from the capstone - v1"` and `"version": "v1"`
- Run the `curl` command 4-5 times in a row and note the `hostname` field changes between the two Pod
  names - proof the Service is actually load-balancing across both replicas, not just hitting one

---

## Part 3: Push the Chart to GitHub (20 min)

ArgoCD works by watching a **real Git repository** - this is the actual GitOps mechanism, not a
simulation of it.

### Step 3.1: Create a new GitHub repository

Go to [github.com/new](https://github.com/new), create a new **public** repository (a private repo
works too, but needs extra ArgoCD credential setup this capstone skips for time). Don't initialize it
with a README - you'll push your existing folder into it.

### Step 3.2: Push your project

```
cd ..
git init
git add .
git commit -m "Initial capstone: app, Dockerfile, Helm chart"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
git push -u origin main
```

Replace `YOUR-USERNAME/YOUR-REPO-NAME` with your actual repository. If prompted for credentials, use a
GitHub Personal Access Token as the password (GitHub no longer accepts your account password directly
for this) - create one under GitHub Settings > Developer Settings > Personal Access Tokens if you don't
have one.

### Verify Part 3

Refresh your GitHub repository page in a browser - you should see `app/`, `helm-chart/`, and everything
inside them, committed and pushed.

---

## Part 4: Install ArgoCD and Connect It to Your Repo (35 min)

### Step 4.1: Install ArgoCD into your minikube cluster

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This takes a few minutes to fully come up. Wait for it:

```
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=180s
```

### Step 4.2: Get the initial admin password

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Copy the printed password - you'll need it in a moment. (This secret only exists until you change the
password for the first time - if you come back to this later and it's gone, that means you already
changed it.)

### Step 4.3: Access the ArgoCD UI

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Leave this running in its own terminal. Open **https://localhost:8080** in your browser - accept the
self-signed certificate warning (this is expected for a local install with no real TLS certificate).
Log in with username `admin` and the password from Step 4.2.

### Step 4.4: Create the Application - point ArgoCD at your GitHub repo

**In VS Code:** create `argocd/application.yaml` (a new top-level folder, separate from `app` and
`helm-chart`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capstone-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
    targetRevision: HEAD
    path: helm-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Replace `YOUR-USERNAME/YOUR-REPO-NAME` with your actual repository** - this is the single most common
mistake in this whole capstone; double check it before continuing.

`syncPolicy.automated` with `selfHeal: true` is what makes this genuinely GitOps, not just a one-time
deploy: ArgoCD will continuously watch your repo AND continuously watch the live cluster state, correcting
either one that drifts from the other, without you running any command.

### Step 4.5: Remove your manual Helm release - let ArgoCD take over from here

```
helm uninstall capstone-app
```

This is a deliberate, clean handoff: from this point forward, ArgoCD - not you running `helm` by hand -
owns this deployment.

### Step 4.6: Apply the Application definition

```
kubectl apply -f argocd/application.yaml
```

### Verify Part 4

Go back to the ArgoCD UI (https://localhost:8080) and refresh - you should see a `capstone-app`
application tile. Click into it - within a minute or two it should show **Synced** and **Healthy**, with
a visual graph of the Deployment, ReplicaSet, Pods, Service, and ConfigMap it created. Confirm with
`kubectl get pods` too - you should see the same 2 Pods as before, but now ArgoCD created them, not your
earlier `helm install`.

---

## Part 5: Prove GitOps Works - Change, Push, Watch It Deploy (30 min)

This is the actual point of the whole capstone - everything before this was setup.

### Step 5.1: Make a real change

**In VS Code:** open `helm-chart/values.yaml` and change:
```yaml
message: "Hello from the capstone - v1"
```
to:
```yaml
message: "Hello from the capstone - v2, deployed via GitOps!"
```

Save.

### Step 5.2: Commit and push - and do NOT touch kubectl or helm

```
git add helm-chart/values.yaml
git commit -m "Update message to v2"
git push
```

That's it. No `kubectl apply`, no `helm upgrade` - if you find yourself wanting to type one of those
commands right now, stop, that would defeat the entire point of this exercise.

### Step 5.3: Watch ArgoCD pick it up

Switch to the ArgoCD UI. Within about 3 minutes (ArgoCD's default polling interval), the `capstone-app`
tile will show **OutOfSync**, then automatically start syncing, then return to **Synced**. Watch the Pods
in the graph - you should see the old ones terminate and new ones start.

If you don't want to wait for the polling interval, force an immediate check from the UI by clicking the
app tile, then the **Refresh** button (the sync itself still respects `automated`, this just makes
ArgoCD check the repo right now instead of waiting for its next scheduled poll).

### Step 5.4: Confirm the new message is actually live

```
curl $(minikube service capstone-app --url)/
```

### Step 5.5: Prove self-healing - manually break something, watch ArgoCD fix it

```
kubectl scale deployment capstone-app --replicas=5
kubectl get pods
```

You just manually changed the cluster to have 5 Pods, bypassing Git entirely. Wait a minute, then check
again:

```
kubectl get pods
```

ArgoCD's `selfHeal: true` should have already noticed the live state (5 replicas) no longer matches what
Git says (2 replicas, from `values.yaml`) and corrected it back down to 2, with no action from you.

### Verify Part 5 (and the whole capstone)

- The `curl` response shows the v2 message, having never run `kubectl apply` or `helm upgrade` yourself
- After manually scaling to 5, the Pod count returned to 2 on its own within about a minute
- You can explain, in your own words, why this qualifies as "GitOps" - specifically: Git is the single
  source of truth, and the cluster continuously reconciles toward whatever Git currently says, in both
  directions (a Git change updates the cluster; a cluster drift gets corrected back toward Git)

---

## Cleanup

```
kubectl delete -f argocd/application.yaml
kubectl delete namespace argocd
helm uninstall capstone-app 2>/dev/null || true
minikube stop
```

Or, to remove everything including the cluster itself:
```
minikube delete
```
