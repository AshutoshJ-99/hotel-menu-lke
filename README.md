# Hotel Menu on Linode LKE (Docker + Kubernetes + GitHub Actions)

This project shows how to deploy a simple **hotel menu website** to **Linode LKE** using:

- Docker
- Kubernetes (LKE)
- Docker Hub
- GitHub Actions (CI/CD)
- Zero-downtime rolling deployments

---

## 1. Prerequisites

You need:

- A **GitHub** account
- A **Docker Hub** account
- A **Linode** account
- On your local machine (or WSL/Ubuntu):
  - `git`
  - `docker`
  - `kubectl`

Example (Ubuntu):

```bash
sudo apt update
sudo apt install -y git curl

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"

# Install kubectl (Linux amd64)
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client
```

Log out & back in so Docker group is applied.

---

## 2. Project Structure

Repo layout:

```text
.
├─ app/
│  └─ index.html
├─ k8s/
│  ├─ deployment.yaml
│  └─ service.yaml
├─ Dockerfile
└─ .github/
   └─ workflows/
      └─ cicd.yaml
```

---

## 3. Static Website (Hotel Menu)

### 3.1 `app/index.html`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>Hotel Menu</title>
  <style>
    body { font-family: Arial; margin: 40px; }
    table { border-collapse: collapse; width: 60%; }
    th, td { border: 1px solid #ccc; padding: 8px; }
    .version { margin-top: 20px; color: #777; }
  </style>
</head>
<body>
  <h1>Hotel Menu</h1>

  <table>
    <tr><th>Item</th><th>Price</th></tr>
    <tr><td>Masala Dosa</td><td>₹80</td></tr>
    <tr><td>Paneer Butter Masala</td><td>₹220</td></tr>
    <tr><td>Tea</td><td>₹15</td></tr>
  </table>

  <div class="version">
    Version: <b>v1</b>
  </div>
</body>
</html>
```

---

## 4. Docker Setup

### 4.1 `Dockerfile`

```dockerfile
FROM nginx:1.27-alpine

# Remove default page
RUN rm -rf /usr/share/nginx/html/*

# Copy our static site
COPY app /usr/share/nginx/html

EXPOSE 80
```

### 4.2 Test locally

```bash
# From repo root
docker build -t hotel-menu:local .
docker run -d --name hotel-menu -p 8080:80 hotel-menu:local
```

Open: `http://localhost:8080`

Stop container when done:

```bash
docker stop hotel-menu && docker rm hotel-menu
```

---

## 5. GitHub Repository

Create a GitHub repo (empty):

- Name: `hotel-menu-lke`
- **Do NOT** initialize with README/License.

Then from project folder:

```bash
git init
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/hotel-menu-lke.git
git add .
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

---

## 6. Docker Hub Image

### 6.1 Create Docker Hub repository

On Docker Hub:

- Create repo: `hotel-menu`
- Visibility: public

### 6.2 Build and push image

```bash
docker build -t <YOUR_DOCKERHUB_USERNAME>/hotel-menu:latest .
docker login    # login with your Docker Hub credentials
docker push <YOUR_DOCKERHUB_USERNAME>/hotel-menu:latest
```

---

## 7. Linode LKE Cluster

1. Go to **Linode Cloud Manager → Kubernetes → Create Cluster**.
2. Choose:
   - Region: closest to you
   - Kubernetes version: latest
   - Node pool: 1 node (2GB) is enough for demo.
3. Create cluster.
4. Click **Download kubeconfig** for this cluster.

Move kubeconfig to your machine (example on WSL/Ubuntu):

```bash
mkdir -p ~/.kube
cp /mnt/c/Users/<YourWindowsUser>/Downloads/kubeconfig.yaml ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

You should see at least one node in `Ready` state.

---

## 8. Kubernetes Manifests

### 8.1 `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotel-menu-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hotel-menu
  template:
    metadata:
      labels:
        app: hotel-menu
    spec:
      containers:
        - name: hotel-menu
          image: <YOUR_DOCKERHUB_USERNAME>/hotel-menu:latest
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

Replace `<YOUR_DOCKERHUB_USERNAME>` with your Docker Hub username.

### 8.2 `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hotel-menu-service
spec:
  type: LoadBalancer
  selector:
    app: hotel-menu
  ports:
    - port: 80
      targetPort: 80
```

### 8.3 First manual apply (optional but useful)

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get svc hotel-menu-service
```

Wait until `EXTERNAL-IP` is assigned (not `<pending>`).  
Open `http://EXTERNAL-IP` to see the site.

---

## 9. GitHub Secrets for CI/CD

Go to:

**Repo → Settings → Secrets and variables → Actions → New repository secret**

Create / set these secrets:

### 9.1 `DOCKERHUB_USERNAME`

```text
<YOUR_DOCKERHUB_USERNAME>
```

### 9.2 `DOCKERHUB_TOKEN`

- Generate on Docker Hub: **Settings → Personal access tokens → Generate**
- Give it **Read & Write** permissions.
- Paste here.

### 9.3 `KUBE_CONFIG_DATA`

- On your local machine:

  ```bash
  cat ~/.kube/config
  ```

- Copy the **entire YAML content** (from `apiVersion: v1` to the end).
- Paste that raw YAML into the secret value (multi-line is OK).

> Note: We store raw kubeconfig (not base64) and write it directly in the workflow.

---

## 10. GitHub Actions Workflow (`.github/workflows/cicd.yaml`)

```yaml
name: Deploy to LKE

on:
  push:
    branches: [ "main" ]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/hotel-menu:latest

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_NAME }}

      - name: Setup kubeconfig
        run: |
          mkdir -p ~/.kube
          printf "%s
" "${KUBE_CONFIG_DATA}" > ~/.kube/config
        env:
          KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}

      - name: Show Kubernetes context
        run: |
          kubectl config get-contexts
          kubectl get nodes

      - name: Apply k8s manifests
        run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml

      - name: Rolling restart deployment
        run: |
          kubectl rollout restart deployment hotel-menu-deployment
          kubectl rollout status deployment hotel-menu-deployment --timeout=120s
```

Once this file is committed and pushed, **any push to `main`** will:

1. Build Docker image
2. Push to Docker Hub
3. Apply Kubernetes manifests
4. Perform a rolling restart (zero downtime)

---

## 11. Zero-Downtime Update Example

To change menu items and deploy with zero downtime:

1. Edit `app/index.html` (update prices or items, change version to `v2`):

   ```html
   <div class="version">
     Version: <b>v2</b>
   </div>
   ```

2. Commit and push:

   ```bash
   git add app/index.html
   git commit -m "Update menu to v2"
   git push
   ```

3. GitHub Actions pipeline will run and update the deployment.
4. During deployment you can keep refreshing `http://EXTERNAL-IP`:
   - Site should never go down.
   - After rollout completes, new menu + version v2 is visible.

---

## 12. Useful kubectl Commands

```bash
# Check nodes
kubectl get nodes

# Check deployments
kubectl get deployments

# Check pods (with status)
kubectl get pods

# Check service and external IP
kubectl get svc
kubectl get svc hotel-menu-service

# See rollout status
kubectl rollout status deployment hotel-menu-deployment

# Describe resources (for debugging)
kubectl describe pod <pod-name>
kubectl describe svc hotel-menu-service
```

---

## 13. Summary

- **Local dev**: simple HTML + Nginx Docker image  
- **Registry**: Docker Hub repo `<YOUR_DOCKERHUB_USERNAME>/hotel-menu`  
- **Cluster**: Linode LKE with 1 small node  
- **Kubernetes**: Deployment (2 replicas) + LoadBalancer Service  
- **CI/CD**: GitHub Actions:
  - build & push image
  - apply manifests
  - rolling restart → **zero downtime**
