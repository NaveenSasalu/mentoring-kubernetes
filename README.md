# Your First Kubernetes App (local first → then k3s in cloud)

A tiny guide to run a **website locally** (WSL2 + Docker), then on **k3s** in the cloud—with **kubectl & Helm from your laptop**.

---

<details>
<summary><strong>0) What you’ll create</strong></summary>

- Local site via **Docker** on **WSL2 (Ubuntu)**
- GitHub repo name-surname building an image via Actions → pushed to **Docker Hub**
- Single-node **k3s** on Hetzner; public HTTPS via **ingress-nginx** + **cert-manager**
- Domain {name}{surname}.com pointing to your server
</details>

---

<details>
<summary><strong>1) Windows prep: install WSL2 & Ubuntu</strong></summary>

**In PowerShell (Admin):**
```powershell
wsl --install -d Ubuntu
```

Reboot if asked → open **Ubuntu** app → create user → update:
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
```
</details>

---

<details>
<summary><strong>2) Install Docker inside WSL2 (Ubuntu)</strong></summary>

```bash
# prerequisites
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Docker repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker run --rm hello-world
```
</details>

---

<details>
<summary><strong>3) Create GitHub repo & minimal website (local)</strong></summary>

- GitHub username: name-surname (e.g., jane-doe)
- Repo: name-surname

**index.html**
```html
<!doctype html>
<html>
  <head><meta charset="utf-8"><title>Hello k3s</title></head>
  <body style="font-family:sans-serif">
    <h1>Hello from Docker on WSL2 👋</h1>
  </body>
</html>
```

**Dockerfile**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

**Run locally:**
```bash
docker build -t local/web:latest .
docker run --rm -p 8080:80 local/web:latest
# open http://localhost:8080
```
</details>

---

<details>
<summary><strong>4) Docker Hub + GitHub Actions (build & push)</strong></summary>

Create an account on **hub.docker.com**.

**.github/workflows/build.yml**
```yaml
name: build-and-push
on:
  push:
    branches: [ main ]
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/name-surname:latest
```

In **Repo → Settings → Secrets and variables → Actions**, add:
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN` (Docker Hub access token)

Push to main → image appears at:
```
docker.io/<DOCKERHUB_USERNAME>/name-surname:latest
```
</details>

---

<details>
<summary><strong>5) Register your domain</strong></summary>

Buy {name}{surname}.com (e.g., via GoDaddy).
You’ll point DNS to your server **after** it’s created.
</details>

---

<details>
<summary><strong>6) Order the server (Hetzner)</strong></summary>

Create a small VM at **Hetzner Cloud** (plans from ~€3.49/mo).
Note the **public IPv4** and ensure **SSH** access.
</details>

---

<details>
<summary><strong>6a) (Optional, recommended) Auto-provision with cloud-init</strong></summary>

When creating the server in Hetzner:
- Add your **SSH key**
- Paste this into the **User data (cloud-init)** box to auto-install k3s, enable UFW, and set kubeconfig perms

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - curl
  - ufw

runcmd:
  # 1) Firewall lockdown
  - ufw default deny incoming
  - ufw default allow outgoing
  - ufw allow 22/tcp      # SSH
  - ufw allow 80/tcp      # HTTP
  - ufw allow 443/tcp     # HTTPS
  - ufw allow 6443/tcp    # K3s API server
  - ufw --force enable

  # 2) K3s install + kubeconfig perms (multi-line script)
  - |
    #!/usr/bin/env bash
    set -e

    IP="$(hostname -I | awk '{print $1}')"
    curl -sfL https://get.k3s.io | \
      INSTALL_K3S_EXEC="server --tls-san ${IP} --disable=traefik" sh -

    chmod 644 /etc/rancher/k3s/k3s.yaml
```

**Notes**
- Traefik is disabled (we’ll install **ingress-nginx**).
- Ports 80/443 (ingress) and 6443 (k8s API) opened.
- After boot, you can **skip step 7 (manual k3s install)** and jump to **step 8/9** to fetch kubeconfig and manage the cluster from your laptop.
</details>

---

<details>
<summary><strong>7) Install k3s on the server (manual)</strong></summary>

> **Skip this if you used step 6a cloud-init** (k3s is already installed).

SSH from your laptop (Windows Terminal/Ubuntu):
```bash
ssh root@<your-server-ip>
```

Install **k3s** (disables Traefik—we’ll use ingress-nginx):
```bash
curl -sfL https://get.k3s.io | sh -s - --disable traefik
# verify
kubectl get nodes
```
> k3s installs kubectl on the server, but we’ll manage the cluster **from your laptop** next.
</details>

---

<details>
<summary><strong>8) Install kubectl & Helm on your laptop (WSL2 Ubuntu)</strong></summary>

**kubectl (stable):**
```bash
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /"  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubectl
kubectl version --client
```

**Helm:**
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
</details>

---

<details>
<summary><strong>8.1) Install k9s (TUI for Kubernetes)</strong></summary>

A handy terminal UI to explore pods/logs/resources.

**Install (WSL2 Ubuntu):**
```bash
# downloads latest Linux amd64 release and puts binary in /usr/local/bin
curl -sL "https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz" \
  | sudo tar zx -C /usr/local/bin k9s
sudo chmod 0755 /usr/local/bin/k9s

# verify
k9s version
```

**Quick use:**
- Launch: `k9s`
- Change namespace: `:ns <name>` (e.g., `:ns ingress-nginx`)
- View logs: select a pod → press `l`
- Describe: `d`
- Quit: `q`
</details>

---

<details>
<summary><strong>9) Configure local kubectl to talk to your k3s</strong></summary>

Copy the k3s kubeconfig **from the server** to your laptop:
```bash
# on your laptop
scp root@<your-server-ip>:/etc/rancher/k3s/k3s.yaml ./k3s.yaml
```

If you used **cloud-init (6a)**, the server cert already includes its IP via `--tls-san`.  
Edit the server address inside k3s.yaml to use the **public IP**:
```bash
# replace 127.0.0.1 with the server public IP
sed -i 's/127.0.0.1/<your-server-ip>/g' k3s.yaml
```

Use it for the current shell:
```bash
export KUBECONFIG=$PWD/k3s.yaml
kubectl get nodes
# should show the k3s node
```
</details>

---

<details>
<summary><strong>9.1) Copy kubeconfig to ~/.kube/config (default path)</summary>

Instead of exporting `KUBECONFIG`, place the file where `kubectl` looks by default.

```bash
# ensure the kube dir exists
mkdir -p ~/.kube

# (optional) backup any existing config
[ -f ~/.kube/config ] && cp ~/.kube/config ~/.kube/config.bak.$(date +%Y%m%d%H%M%S)

# copy and lock down permissions
cp ./k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config

# test
kubectl get nodes
```
</details>

---

<details>
<summary><strong>10) Install ingress-nginx (from your laptop via Helm)</strong></summary>

Expose ports 80/443 via hostPorts (single-node VM):
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.hostPort.enabled=true \
  --set controller.kind=DaemonSet

kubectl -n ingress-nginx get pods
```
</details>

---

<details>
<summary><strong>11) Install cert-manager (Let’s Encrypt) from your laptop</strong></summary>

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace
```

Create a **ClusterIssuer** (replace your email):

**cluster-issuer.yaml**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply:
```bash
kubectl apply -f cluster-issuer.yaml
```
</details>

---

<details>
<summary><strong>12) Point your domain to the server</strong></summary>

At your registrar (e.g., GoDaddy), set DNS:
- A record: `@` → `<your-server-ip>`
- (optional) A record: `www` → `<your-server-ip>`

Wait a few minutes for propagation.
</details>

---

<details>
<summary><strong>13) Deploy your web app to k3s (from laptop)</strong></summary>

Replace `<DOCKERHUB_USERNAME>` and `<YOUR_DOMAIN>`.

**app.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app: web }
spec:
  replicas: 1
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: <DOCKERHUB_USERNAME>/name-surname:latest
        ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-http
    kubernetes.io/ingress.class: nginx
spec:
  tls:
    - hosts: [ "<YOUR_DOMAIN>" ]
      secretName: web-tls
  rules:
    - host: <YOUR_DOMAIN>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Apply & check:
```bash
kubectl apply -f app.yaml
kubectl get pods,svc,ingress
# open https://<YOUR_DOMAIN>
```
</details>

---

<details>
<summary><strong>Troubleshooting quickies</strong></summary>

- DNS: confirm A record points to the server IP
- Ingress: `kubectl describe ingress web`
- Nginx: `kubectl -n ingress-nginx get pods`
- Certs: `kubectl -n cert-manager get challenges,orders,certificates`
- Firewall: `ufw status` (80/443/6443 must be allowed)
</details>

---

## Done 🎉
- Local dev: **Docker on WSL2** → http://localhost:8080  
- CI/CD: **GitHub Actions** → **Docker Hub**  
- Cloud: **k3s** (managed from your laptop with **kubectl**, **Helm**, **k9s**), **ingress-nginx**, **cert-manager** → HTTPS website
