# CI/CD → Kubernetes Practical Lab

Goal: edit code in VS Code → `git push` → GitHub Actions builds a Docker image
and deploys it to a Kubernetes cluster **running on your own laptop**.

## Why a self-hosted runner?

GitHub's normal ("cloud") Actions runners live on GitHub's servers — they have
no network path to a cluster sitting on your laptop. To let a workflow
`kubectl apply` against your local `kind`/`minikube` cluster, you install a
tiny agent (a "self-hosted runner") on your laptop. GitHub then dispatches
the `deploy` job to that agent instead of to the cloud, so it runs locally
and can reach your cluster. This is the standard pattern for exactly this lab.

---

## Step 0 — Install prerequisites on your laptop

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (or Docker Engine)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) (Kubernetes-in-Docker) — easiest for a laptop lab
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [Git](https://git-scm.com/downloads) + [VS Code](https://code.visualstudio.com/)
- A GitHub account

Verify:
```bash
docker --version
kind --version
kubectl version --client
git --version
```

## Step 1 — Create the local Kubernetes cluster

```bash
kind create cluster --name lab
kubectl cluster-info --context kind-lab
```

## Step 2 — Create the GitHub repo and push this code

1. On GitHub, create a new empty repo, e.g. `k8s-lab-app`.
2. In this project folder:
```bash
cd k8s-lab
git init
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/k8s-lab-app.git
git add .
git commit -m "Initial commit: app + k8s manifests + CI/CD workflow"
git branch -M main
git push -u origin main
```

3. **Edit `k8s/deployment.yaml`** and replace `YOUR_GITHUB_USERNAME` with your
   actual GitHub username (this is the GHCR image path).

## Step 3 — Make your repo's GitHub Container Registry packages public (or set up pull secret)

By default images pushed to `ghcr.io` under your repo are private. Easiest
for a lab: after the first successful build, go to
`github.com/YOUR_USERNAME?tab=packages` → the `k8s-lab-app` package →
Package settings → Change visibility → **Public**. This avoids needing an
`imagePullSecret` on your local cluster.

## Step 4 — Install a self-hosted GitHub Actions runner on your laptop

1. On GitHub: repo → **Settings → Actions → Runners → New self-hosted runner**.
2. Pick your OS and follow the exact commands shown (they include a
   repo-specific registration token), roughly:
```bash
mkdir actions-runner && cd actions-runner
# download the package GitHub shows you for your OS
tar xzf ./actions-runner-*.tar.gz
./config.sh --url https://github.com/YOUR_GITHUB_USERNAME/k8s-lab-app --token YOUR_TOKEN
./run.sh
```
3. Leave that terminal running (or install it as a background service using
   `./svc.sh install && ./svc.sh start` on Linux/macOS, or the Windows
   equivalent). This laptop is now listed as an online runner for the repo,
   and it already has `kubectl` configured for your `kind` cluster since you
   set that up in Step 1.

## Step 5 — Trigger the pipeline

Make a small change in VS Code, e.g. edit the message in `app/index.js`, then:
```bash
git add .
git commit -m "test: trigger pipeline"
git push
```

Go to your repo's **Actions** tab and watch:
- `build-and-push` job runs on GitHub's cloud runner → builds Docker image → pushes to `ghcr.io`
- `deploy` job runs on **your laptop's runner** → `kubectl apply` + `kubectl set image` → rolls out the new pod

## Step 6 — Verify it worked

```bash
kubectl get pods
kubectl get svc k8s-lab-app-svc
curl http://localhost:30080
```
(`kind` maps the NodePort automatically on Linux; on macOS/Windows Docker
Desktop, you may need `kubectl port-forward svc/k8s-lab-app-svc 8080:80`
and then `curl http://localhost:8080` instead — kind's NodePort isn't
always reachable directly on those hosts.)

You should see JSON with your updated message and a timestamp.

---

## Repo layout

```
k8s-lab/
├── app/
│   ├── index.js          # tiny Express app
│   ├── package.json
│   └── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── .github/workflows/
│   └── ci-cd.yaml        # build -> push GHCR -> deploy via self-hosted runner
└── README.md
```

## Swapping the app language

The app is just Node/Express to keep it minimal, but nothing else in the
pipeline cares about language — only `app/Dockerfile` needs to change (base
image + build/run commands). The workflow, manifests, and runner setup stay
identical.

## Next things to try in this lab

- Add a `staging`/`production` branch split with two Deployments.
- Add a unit-test job before `build-and-push` that fails the pipeline on test failure.
- Swap `kubectl set image` for a Helm chart or Kustomize overlay.
- Try ArgoCD for GitOps-style deployment instead of a push-based `kubectl apply`.
