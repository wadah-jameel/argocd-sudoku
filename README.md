# argocd-sudoku
Guide to deploying a Sudoku game on Kubernetes via ArgoCD

Prerequisites

- A Kubernetes cluster (local: minikube/kind, or cloud: EKS/GKE/AKS)
- kubectl installed and configured
- git installed
- A GitHub/GitLab account
- Docker installed (to build your image)
- A Docker Hub or container registry account


Step 1: Containerize Your Website

## First, package your website into a Docker image.

### Create a Dockerfile in your website's root directory:

## Build and push the image:

```bash
docker build -t your-dockerhub-username/sudoku:v1 .
docker push your-dockerhub-username/sudoku:v1
```

---

## Step 2: Create a Git Repository for Kubernetes Manifests

ArgoCD follows the **GitOps** model — your cluster state lives in Git. Keep your app code and K8s manifests in **separate repos** (best practice).

Create a new repo called `sudoku` on GitHub, then structure it like this:
```
my-website-k8s/
└── manifests/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml      # optional
```

Step 3: Write Kubernetes Manifests

## manifests/deployment.yaml

## manifests/service.yaml

## Commit and push both files to GitHub:

```bash
git add .
git commit -m "Add initial K8s manifests"
git push origin main
```
Step 4: Install ArgoCD on Your Cluster

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be running
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

Access the ArgoCD UI:

```bash
# Port-forward to access locally
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then open https://localhost:8080 in your browser.

## Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```
### Login with username admin and the password above.

Step 5: Install ArgoCD CLI (Optional but Recommended)

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login localhost:8080 --username admin --password <your-password> --insecure
```


Step 6: Connect Your Git Repo to ArgoCD

## If your repo is public, ArgoCD can access it without credentials. For a private repo:

```bash
argocd repo add https://github.com/your-username/sudoku \
  --username your-github-username \
  --password your-github-token
```
## Or via the UI: Settings → Repositories → Connect Repo.

Step 7: Create the ArgoCD Application

### This is the core step — it tells ArgoCD which Git repo to watch and where to deploy.

## Option A — via CLI:

```bash
argocd app create sudoku \
  --repo https://github.com/your-username/sudoku \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace sudoku \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Option B — via YAML (recommended for GitOps):

### Create argocd-app.yaml

### Apply it:

```bash
kubectl apply -f argocd-app.yaml
```

Step 8: Sync and Verify the Deployment

### Check app status:

```bash
argocd app get sudoku
```

### Manually trigger a sync (if not automated):

```bash
argocd app sync sudoku
```
### Verify pods and service are running:

```bash
kubectl get pods -n sudoku
kubectl get svc -n sudoku
```

Step 9: Access Your Website

### On minikube:

```bash
minikube service sudoku-svc
```
### On a cloud cluster (LoadBalancer):

```bash
kubectl get svc sudoku-svc
# Use the EXTERNAL-IP shown in the output
```

Step 10: Deploy Updates (The GitOps Flow)

### This is where ArgoCD shines. To deploy a new version:

## 1- Build and push a new Docker image:

```bash
docker build -t your-dockerhub-username/sudoku:v2 .
docker push your-dockerhub-username/sudoku:v2
```

## 2- Update deployment.yaml in your Git repo — change the image tag from v1 to v2:

```bash
image: your-dockerhub-username/sudoku:v2
```

## 3- Commit and push:

```bash
git commit -am "Update website to v2"
git push origin main
```

4. ArgoCD detects the change and **automatically syncs** — no `kubectl apply` needed.

---

## Full Flow Summary
```
Your Code → Docker Image → Push to Registry
                                  ↓
           Update image tag in Git manifest repo
                                  ↓
              ArgoCD detects Git diff → Auto Sync
                                  ↓
         Kubernetes applies new Deployment → Rolling Update
```

## Pro Tips

```markdown
* Use Helm charts instead of raw manifests for more complex apps — ArgoCD supports them natively.
* Use image updater (argocd-image-updater) to automatically bump image tags in Git when you push a new image to your registry.
* Set resource limits in your Deployment to prevent pods from consuming too many cluster resources.
* Use namespaces to separate environments (e.g., dev, staging, prod) within the same cluster.
* Never edit resources directly with kubectl when using ArgoCD — always change Git and let ArgoCD sync. Manual changes will be reverted if selfHeal: true is set.
```
