# gitops-demo

A minimal Flask app with a full local Kubernetes setup and GitHub Actions CI/CD pipeline.

---

## Project Structure

```
k8s-python-app/
├── app/
│   ├── main.py            # Flask application
│   ├── requirements.txt
│   └── Dockerfile         # Multi-stage, non-root build
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-configmap.yaml
│   ├── 02-deployment.yaml  # Image tag updated by CI/CD
│   ├── 03-service.yaml
│   ├── 04-ingress.yaml
│   └── 05-hpa.yaml
└── .github/
    └── workflows/
        └── ci-cd.yaml
```

---

## Local Setup (minikube)

### 1. Prerequisites
```bash
brew install minikube kubectl
# OR on Linux:
# curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

### 2. Start minikube
```bash
minikube start --cpus=2 --memory=4g
minikube addons enable ingress
minikube addons enable metrics-server   # needed for HPA
```

### 3. Build image and push to GitHub Container Registry (ghcr.io)
```bash

docker login ghcr.io -u YOUR_USERNAME # provide the access token you created as the password

docker build -t ghcr.io/YOUR_GITHUB_USERNAME/python-app:latest ./app

# On github, ensure you've set the package as public for the cluster to be able to pull the image

docker push ghcr.io/YOUR_GITHUB_USERNAME/python-app:latest
```

### 4. Update the image name
Edit `k8s/02-deployment.yaml` and replace `YOUR_GITHUB_USERNAME` with your actual GitHub username.

### 5. Deploy
```bash
kubectl apply -f k8s/
```

### 6. Access the app
```bash
# Add to /etc/hosts
echo "$(minikube ip) python-app.local" | sudo tee -a /etc/hosts

# Open in browser or curl
curl http://python-app.local
```

### 7. Useful commands
```bash
# Watch pods
kubectl get pods -n python-app -w

# Logs
kubectl logs -n python-app -l app=python-app -f

# Port-forward (alternative to ingress)
kubectl port-forward svc/python-app 8080:80 -n python-app

# Scale manually
kubectl scale deployment python-app --replicas=3 -n python-app

# Rollback
kubectl rollout undo deployment/python-app -n python-app
```

---

## GitHub Actions Pipeline

### Pipeline Stages
| Job | Trigger | What it does |
|-----|---------|--------------|
| `test` | Every push/PR | Lints with flake8, runs pytest |
| `build-push` | Push to `main` | Builds multi-arch image, pushes to GHCR |
| `update-manifests` | After build | Updates `k8s/02-deployment.yaml` with new image tag, commits back |
| `deploy` | After manifest update | Applies all manifests, waits for rollout, auto-rollbacks on failure |

### Required GitHub Secrets
Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|--------|-------|
| `KUBECONFIG` | Base64-encoded kubeconfig: `cat ~/.kube/config \| base64` |

> **Note:** `GITHUB_TOKEN` is provided automatically by GitHub Actions — no setup needed for pushing to GHCR.

### Image Tagging Strategy
- `main-<short-sha>` — every push to main (e.g. `main-abc1234`)
- `latest` — floating tag always pointing to the newest main build
- `v1.2.3` / `v1.2` — when you push a git tag

---

## Connecting a Real Cluster (e.g. EKS / GKE / AKS)

1. Get your cluster's kubeconfig.
2. Base64 encode it: `cat kubeconfig.yaml | base64 | pbcopy`
3. Paste into GitHub secret `KUBECONFIG`.
4. The pipeline will deploy automatically on every push to `main`.

---

## App Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /` | Returns hostname, version, environment |
| `GET /health` | Liveness probe |
| `GET /ready` | Readiness probe |