# Module 00 — Local Setup for macOS (CKA Practice Environment)

> Install and verify every tool you need before touching Module 01. A properly configured local environment is what lets you run every lab in this course without a cloud account.

---

## What You Will Install

| Tool | Purpose | Used In |
|------|---------|---------|
| Homebrew | macOS package manager | Everything |
| Docker Desktop | Container runtime + Docker CLI | Module 01+ |
| kubectl | Kubernetes CLI | All modules |
| minikube | Local single/multi-node cluster | All modules |
| kind | Kubernetes-in-Docker (alternative cluster) | Module 03 |
| Helm | Kubernetes package manager | Module 11 |
| k9s | Terminal UI for cluster inspection | All modules |
| multipass | Lightweight Linux VMs (kubeadm labs) | Module 03 |
| etcdctl | CLI for etcd (backup/restore) | Module 03, 10 |
| htop | Process monitor | Module 01 |
| jq | JSON processor (kubectl output parsing) | All modules |
| tree | Directory tree viewer | Module 11 |

---

## Step 1 — Homebrew

Homebrew is the prerequisite for everything else.

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# For Apple Silicon (M1/M2/M3/M4) — add Homebrew to PATH
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Verify
brew --version
# Expected: Homebrew 4.x.x
```

---

## Step 2 — Docker Desktop

Docker Desktop provides the container runtime that minikube and kind use.

```bash
# Install Docker Desktop
brew install --cask docker

# After installation, launch it from Applications
open -a Docker
```

**Wait ~60 seconds** for Docker to fully start, then verify:

```bash
docker --version
# Expected: Docker version 27.x.x

docker info | grep "Server Version"
# Expected: Server Version: 27.x.x

# Quick smoke test
docker run --rm hello-world
# Expected: "Hello from Docker!"
```

### Docker Desktop Settings (Important)

Open Docker Desktop → Settings → Resources and set:
- **CPUs:** at least 4
- **Memory:** at least 8 GB
- **Disk image size:** at least 60 GB

These resources are shared with minikube and kind clusters.

---

## Step 3 — kubectl

The primary tool for interacting with Kubernetes clusters.

```bash
brew install kubectl

# Verify
kubectl version --client
# Expected: Client Version: v1.31.x

# Enable shell completion (add to ~/.zshrc or ~/.bashrc)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
echo 'complete -F __start_kubectl k' >> ~/.zshrc
source ~/.zshrc
```

---

## Step 4 — minikube

Your primary local cluster. Used in almost every lab in this course.

```bash
brew install minikube

# Verify
minikube version
# Expected: minikube version: v1.34.x
```

### Start Your First Cluster

```bash
# Start with enough resources for labs
minikube start \
  --cpus=4 \
  --memory=8192 \
  --disk-size=30g \
  --driver=docker

# This takes 2-5 minutes on first run (downloads the node image)
# Expected output ends with: Done! kubectl is now configured to use "minikube"
```

### Verify the Cluster

```bash
# Check node is Ready
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.31.x

# Check all system pods are Running
kubectl get pods -n kube-system
# All pods should be Running or Completed — none should be Pending or Error

# Check cluster info
kubectl cluster-info
# Expected: Kubernetes control plane is running at https://127.0.0.1:PORT

# Stop cluster when done (saves resources)
minikube stop
```

### Essential minikube Commands

```bash
minikube start                    # start cluster
minikube stop                     # stop cluster (state preserved)
minikube delete                   # delete cluster entirely
minikube status                   # show status
minikube ip                       # get node IP
minikube ssh                      # SSH into the node
minikube dashboard                # open web UI in browser
minikube addons list              # available addons
minikube addons enable ingress    # enable ingress controller
minikube addons enable metrics-server  # enable metrics-server

# Multi-node cluster (used in Module 05)
minikube start --nodes=2 --cpus=2 --memory=2048
```

---

## Step 5 — kind (Kubernetes in Docker)

Alternative to minikube. Useful for multi-node setups and Module 03 kubeadm practice.

```bash
brew install kind

# Verify
kind version
# Expected: kind v0.25.x

# Create a single-node cluster
kind create cluster --name cka-practice

# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name cka-practice
```

### kind Multi-Node Config (for kubeadm practice context)

```bash
# Create a config file
cat > /tmp/kind-multinode.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --name multi --config /tmp/kind-multinode.yaml
kubectl get nodes
# Shows: 1 control-plane + 2 workers

kind delete cluster --name multi
```

---

## Step 6 — Helm

Kubernetes package manager. Required for Module 11 and exam tasks.

```bash
brew install helm

# Verify
helm version
# Expected: version.BuildInfo{Version:"v3.17.x"...}

# Add the Bitnami chart repository (used in all Helm labs)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
# Expected: bitnami  https://charts.bitnami.com/bitnami
```

---

## Step 7 — k9s (Highly Recommended)

Terminal dashboard for Kubernetes — faster than kubectl for browsing resources.

```bash
brew install k9s

# Verify
k9s version

# Launch (requires a running cluster)
minikube start
k9s
```

### k9s Key Bindings

```
:pods          → view pods
:svc           → view services
:deploy        → view deployments
:ns            → view namespaces
:nodes         → view nodes
:pv            → view persistent volumes
0              → show all namespaces
/              → filter/search
d              → describe resource
l              → view logs
e              → edit resource
ctrl+d         → delete resource
esc            → go back
q              → quit
```

---

## Step 8 — multipass (for kubeadm labs)

Creates real Ubuntu VMs on macOS. Used in Module 03 for kubeadm cluster setup labs.

```bash
brew install --cask multipass

# Verify
multipass version
# Expected: multipass  1.x.x

# Quick test
multipass launch --name test-vm 22.04
multipass list
multipass shell test-vm
  # inside VM:
  uname -a    # should show Ubuntu Linux
  exit
multipass delete test-vm && multipass purge
```

---

## Step 9 — etcdctl

CLI for managing etcd directly. Critical for Module 03 (backup/restore) and Module 10 (troubleshooting).

```bash
# Install via Homebrew
brew install etcd    # installs etcd + etcdctl

# Verify
etcdctl version
# Expected: etcdctl version: 3.5.x

# Set API version permanently (add to ~/.zshrc)
echo 'export ETCDCTL_API=3' >> ~/.zshrc
source ~/.zshrc
```

---

## Step 10 — Supporting Tools

```bash
# htop — process monitor (Module 01)
brew install htop
htop --version

# jq — JSON processor (parsing kubectl output)
brew install jq
echo '{"name":"k8s","version":1}' | jq '.name'

# tree — directory viewer (Module 11 chart structure)
brew install tree
tree --version

# wget — download tool (some labs use it)
brew install wget
wget --version
```

---

## Step 11 — Shell Configuration

Add all of this to your `~/.zshrc` for a proper CKA practice environment:

```bash
cat >> ~/.zshrc << 'EOF'

# ===== CKA Practice Environment =====

# kubectl autocomplete
source <(kubectl completion zsh)

# Aliases
alias k=kubectl
alias kn='kubectl -n'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kx='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
complete -F __start_kubectl k

# etcdctl API version
export ETCDCTL_API=3

# kubectl output editor (choose one)
export KUBE_EDITOR=nano    # easier for beginners
# export KUBE_EDITOR=vim   # faster once comfortable

# Show current kube context in prompt (optional but useful)
parse_kube_ctx() {
  ctx=$(kubectl config current-context 2>/dev/null)
  ns=$(kubectl config view --minify --output 'jsonpath={..namespace}' 2>/dev/null)
  ns=${ns:-default}
  echo "[${ctx}|${ns}]"
}
# Uncomment to add to PS1:
# PS1='$(parse_kube_ctx) '$PS1

# ===== End CKA =====
EOF

source ~/.zshrc
```

---

## Step 12 — VS Code Extensions (Optional but Recommended)

If you use VS Code for editing YAML files:

```bash
# Install VS Code CLI tool first (from VS Code: Cmd+Shift+P → Shell Command: Install 'code' in PATH)

# Kubernetes extension
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools

# YAML language support (schema validation for k8s YAML)
code --install-extension redhat.vscode-yaml

# Verify
code --list-extensions | grep -E "kubernetes|yaml"
```

---

## Full Installation Verification

Run this script to verify everything is working:

```bash
#!/bin/bash
echo "====== CKA Tool Check ======"

check() {
  if command -v $1 &>/dev/null; then
    echo "✓ $1 — $($2 2>&1 | head -1)"
  else
    echo "✗ $1 — NOT FOUND"
  fi
}

check "brew"       "brew --version"
check "docker"     "docker --version"
check "kubectl"    "kubectl version --client --short 2>/dev/null || kubectl version --client"
check "minikube"   "minikube version"
check "kind"       "kind version"
check "helm"       "helm version --short"
check "k9s"        "k9s version"
check "multipass"  "multipass version"
check "etcdctl"    "etcdctl version"
check "htop"       "htop --version"
check "jq"         "jq --version"
check "tree"       "tree --version"
check "wget"       "wget --version"

echo ""
echo "====== Cluster Check ======"
if minikube status 2>/dev/null | grep -q "Running"; then
  echo "✓ minikube cluster is running"
  kubectl get nodes
else
  echo "- minikube is stopped (run: minikube start)"
fi
```

Save as `~/check-cka-env.sh`, then:
```bash
chmod +x ~/check-cka-env.sh
~/check-cka-env.sh
```

---

## Lab 00-A — End-to-End Smoke Test

Once all tools are installed, run this to confirm everything works together:

```bash
# 1. Start minikube
minikube start --cpus=4 --memory=8192 --driver=docker

# 2. Confirm cluster is up
kubectl get nodes
kubectl get pods -n kube-system

# 3. Run a pod
kubectl run smoke-test --image=nginx:alpine --port=80
kubectl wait --for=condition=Ready pod/smoke-test --timeout=60s
kubectl get pod smoke-test

# 4. Port-forward and test
kubectl port-forward pod/smoke-test 8080:80 &
PF_PID=$!
sleep 2
curl -s http://localhost:8080 | grep -o "<title>.*</title>"
# Expected: <title>Welcome to nginx!</title>
kill $PF_PID

# 5. Test kubectl explain (API docs access)
kubectl explain pod.spec.containers.resources | head -10

# 6. Test Helm
helm repo update
helm search repo bitnami/nginx | head -3

# 7. Test etcdctl (minikube has etcd inside the node)
minikube ssh -- sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
# Expected: https://127.0.0.1:2379 is healthy

# 8. Cleanup
kubectl delete pod smoke-test
echo "All checks passed — environment is ready!"
```

---

## Lab 00-B — Enable Addons for Lab Coverage

Enable all minikube addons needed across this course:

```bash
minikube start

# Metrics server — needed for kubectl top (Module 09)
minikube addons enable metrics-server

# Ingress controller — needed for Module 06
minikube addons enable ingress

# Ingress DNS — resolve *.local in browser (optional)
minikube addons enable ingress-dns

# Dashboard — optional but useful for visualizing objects
minikube addons enable dashboard

# Verify
minikube addons list | grep -E "enabled|metrics|ingress|dashboard"

# Wait for ingress controller to be ready
kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/component=controller \
  -n ingress-nginx --timeout=120s

echo "Addons ready"
```

---

## Lab 00-C — kubeconfig & Context Basics

```bash
# 1. View your kubeconfig
kubectl config view

# 2. See current context
kubectl config current-context
# Expected: minikube

# 3. List all contexts
kubectl config get-contexts

# 4. Create a test namespace and switch to it
kubectl create namespace cka-practice
kubectl config set-context --current --namespace=cka-practice

# Verify — now kubectl commands default to cka-practice
kubectl get pods    # shows pods in cka-practice namespace

# 5. Switch back to default
kubectl config set-context --current --namespace=default

# 6. Inspect kubeconfig file directly
cat ~/.kube/config | head -30

kubectl delete namespace cka-practice
```

---

## Troubleshooting Common Setup Issues

### minikube fails to start

```bash
# Check Docker is running
docker info

# Reset minikube
minikube delete --all --purge
minikube start --driver=docker --cpus=4 --memory=8192

# If using Apple Silicon and getting issues
minikube start --driver=docker --container-runtime=containerd
```

### kubectl can't connect to cluster

```bash
# Check minikube is running
minikube status

# Refresh kubeconfig
minikube update-context

# Verify connection
kubectl cluster-info
```

### Homebrew install fails (permissions)

```bash
# Fix Homebrew permissions
sudo chown -R $(whoami) /usr/local/lib/pkgconfig
brew doctor
```

### Not enough resources

```bash
# Check how much memory/CPU you have
sysctl -n hw.memsize | awk '{print $0/1024/1024/1024 " GB"}'
sysctl -n hw.logicalcpu

# Start minikube with less resources if needed (minimum for labs)
minikube start --cpus=2 --memory=4096 --driver=docker
```

### Docker Desktop conflicts with VirtualBox/Parallels

```bash
# Explicitly set the driver
minikube start --driver=docker

# Or use hyperkit (macOS native, faster than docker driver)
brew install --cask hyperkit
minikube start --driver=hyperkit --cpus=4 --memory=8192
```

---

## Quick Reference — Start/Stop Workflow

```bash
# Morning: start your lab environment
minikube start
kubectl get nodes    # verify Ready

# Evening: stop to free resources
minikube stop
# Cluster state is preserved — pods/configs survive stop/start

# Full reset (start fresh)
minikube delete
minikube start --cpus=4 --memory=8192 --driver=docker
```

---

## Checklist

- [ ] Homebrew installed and `brew doctor` shows no critical issues
- [ ] Docker Desktop running and `docker run hello-world` succeeds
- [ ] `kubectl` installed and autocomplete works with `k` alias
- [ ] `minikube start` produces a Ready node (`kubectl get nodes`)
- [ ] `kind` installed and can create a cluster
- [ ] `helm` installed and bitnami repo is configured
- [ ] `k9s` launches and shows cluster resources
- [ ] `multipass` installed (needed for Module 03 kubeadm labs)
- [ ] `etcdctl version` works and `ETCDCTL_API=3` is set in shell
- [ ] `jq`, `htop`, `tree`, `wget` installed
- [ ] Shell aliases (`k`, `kg`, `kd`, `kl`) working
- [ ] Smoke test lab (Lab 00-A) completes without errors
- [ ] minikube addons: `metrics-server` and `ingress` enabled

---

**Next:** [Module 01 — Linux & Container Fundamentals](./Module-01-Linux-Container-Fundamentals.md)
