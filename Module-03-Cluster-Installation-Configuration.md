# Module 03 — Cluster Installation & Configuration

> This domain carries major exam weight. Know kubeadm, etcd backup/restore, upgrades, and static pods cold.

---

## 3.1 kubeadm Cluster Setup

### What kubeadm Does

```
kubeadm init  → bootstraps control plane
kubeadm join  → adds worker nodes
kubeadm reset → tears down cluster
kubeadm token → manages bootstrap tokens
kubeadm certs → manages certificates
```

### kubeadm init Flow

```
1. Pre-flight checks (ports, OS, runtime)
2. Generate CA certificates (in /etc/kubernetes/pki/)
3. Write kubeconfig files (/etc/kubernetes/*.conf)
4. Generate static pod manifests (/etc/kubernetes/manifests/)
5. Wait for control plane to start
6. Bootstrap RBAC rules
7. Install CoreDNS and kube-proxy add-ons
8. Print kubeadm join command
```

---

## 3.2 Multi-Node Cluster with kubeadm (Linux VMs)

> For macOS you need Linux VMs. Use multipass (free, lightweight) or the Lima-based approach.

### Setup on macOS — Using Multipass

```bash
# Install multipass
brew install --cask multipass

# Create control plane VM
multipass launch --name master --cpus 2 --memory 4G --disk 20G 22.04

# Create worker VMs
multipass launch --name worker1 --cpus 2 --memory 2G --disk 20G 22.04
multipass launch --name worker2 --cpus 2 --memory 2G --disk 20G 22.04

# List VMs
multipass list

# Get IPs
multipass info master
multipass info worker1
```

### Step 1 — Install Prerequisites (run on ALL nodes)

```bash
# SSH into each node
multipass shell master   # repeat for worker1, worker2

# Disable swap (required by Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set network sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install containerd
sudo apt-get update && sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable kubelet
```

### Step 2 — Initialize Control Plane (master only)

```bash
# On master node:
MASTER_IP=$(hostname -I | awk '{print $1}')

sudo kubeadm init \
  --apiserver-advertise-address=${MASTER_IP} \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=1.31.0

# After init completes, setup kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Verify control plane
kubectl get nodes
kubectl get pods -n kube-system

# SAVE THE JOIN COMMAND printed by kubeadm init
# It looks like: kubeadm join 192.168.x.x:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx
```

### Step 3 — Join Worker Nodes

```bash
# On each worker node (use the command printed by kubeadm init):
sudo kubeadm join 192.168.x.x:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# If token expired (24h TTL), regenerate on master:
kubeadm token create --print-join-command
```

### Step 4 — Verify Cluster

```bash
# On master:
kubectl get nodes -o wide
# Should show all nodes as Ready

kubectl get pods -n kube-system
# All pods should be Running

kubectl run test --image=nginx:alpine
kubectl get pod test
kubectl delete pod test
```

---

## 3.3 High Availability Basics

### HA Control Plane

```
             ┌─────────────────┐
             │  Load Balancer  │  ← HAProxy / keepalived / cloud LB
             │  (VIP: port 6443)│
             └────────┬────────┘
              ┌───────┼───────┐
              ▼       ▼       ▼
         Master1   Master2   Master3
           (etcd)   (etcd)   (etcd)
             └───────┴───────┘
               etcd cluster (3 nodes)
```

```bash
# HA init (stacked etcd)
sudo kubeadm init \
  --control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Join additional control plane nodes
sudo kubeadm join LOAD_BALANCER_DNS:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key>
```

---

## 3.4 etcd Backup and Restore (CRITICAL for exam)

### Backup etcd

```bash
# Install etcdctl
export ETCD_VER=v3.5.9
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz
tar xzvf /tmp/etcd.tar.gz -C /tmp/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/

# Take a snapshot
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db \
  --write-out=table
```

### Restore etcd (exam scenario)

```bash
# Step 1: Stop kube-apiserver (move manifest out)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Step 2: Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 3: Update etcd manifest to use new data dir
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' \
  /etc/kubernetes/manifests/etcd.yaml

# Also update hostPath in volumes section

# Step 4: Move apiserver back
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Step 5: Wait for cluster to recover
kubectl get nodes    # may take 1-2 minutes
```

---

## 3.5 Cluster Upgrade with kubeadm

### Upgrade Process (Exam: drain → upgrade control plane → upgrade workers)

```bash
# ==============================
# On CONTROL PLANE NODE
# ==============================

# Step 1: Check available versions
apt-cache madison kubeadm | head -5

# Step 2: Unhold and upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.32.0-1.1
sudo apt-mark hold kubeadm

# Step 3: Plan the upgrade
sudo kubeadm upgrade plan

# Step 4: Apply the upgrade
sudo kubeadm upgrade apply v1.32.0

# Step 5: Drain the control plane node
kubectl drain master --ignore-daemonsets --delete-emptydir-data

# Step 6: Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo apt-mark hold kubelet kubectl

# Step 7: Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Step 8: Uncordon the node
kubectl uncordon master

# Verify
kubectl get nodes
# Control plane should show v1.32.0

# ==============================
# On EACH WORKER NODE
# ==============================

# From master: drain the worker
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data

# SSH into worker1 then:
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get install -y kubeadm=1.32.0-1.1
sudo kubeadm upgrade node

sudo apt-get install -y kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Back on master: uncordon
kubectl uncordon worker1
kubectl get nodes    # verify worker1 shows v1.32.0
```

---

## 3.6 Static Pods

- Pods managed directly by kubelet, NOT the API server
- Defined in `/etc/kubernetes/manifests/`
- kubelet watches this directory and creates/removes pods automatically
- Control plane components (apiserver, etcd, etc.) ARE static pods
- Identified by the node name suffix: `kube-apiserver-master`

```bash
# Create a static pod
multipass shell master

sudo cat > /etc/kubernetes/manifests/my-static-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
  namespace: default
spec:
  containers:
  - name: static-nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF

# kubelet will pick it up automatically
# Back on your Mac:
kubectl get pod my-static-pod-master -n default

# Delete: remove the manifest file (NOT kubectl delete)
multipass shell master
sudo rm /etc/kubernetes/manifests/my-static-pod.yaml

# Pod will be gone
kubectl get pod    # my-static-pod-master is gone
```

---

## 3.7 Certificate Management

```bash
# View certificate expiry
sudo kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Renew specific cert
sudo kubeadm certs renew apiserver

# View a certificate
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 "Validity"

# Certificate locations
/etc/kubernetes/pki/
├── apiserver.crt / apiserver.key
├── apiserver-kubelet-client.crt / .key
├── ca.crt / ca.key              ← Root CA
├── etcd/
│   ├── ca.crt / ca.key
│   └── server.crt / server.key
└── front-proxy-ca.crt / .key
```

---

## Lab 03-A — etcd Backup/Restore on Minikube

```bash
# 1. SSH into minikube node
minikube ssh

# 2. Check etcd is running
sudo crictl ps | grep etcd

# 3. Set etcdctl variables
ETCD_CERT=/etc/kubernetes/pki/etcd/server.crt
ETCD_KEY=/etc/kubernetes/pki/etcd/server.key
ETCD_CA=/etc/kubernetes/pki/etcd/ca.crt

# 4. Check etcd health
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=$ETCD_CA \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  endpoint health

# 5. Create some test data first
exit
kubectl create namespace backup-test
kubectl create deployment test-app --image=nginx:alpine -n backup-test
kubectl get all -n backup-test

# 6. Take snapshot
minikube ssh
sudo ETCDCTL_API=3 etcdctl snapshot save /tmp/cluster-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 7. Verify snapshot
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/cluster-backup.db --write-out=table
exit
```

---

## Lab 03-B — Static Pod Creation

```bash
# 1. SSH into minikube
minikube ssh

# 2. View existing static pod manifests
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml  etcd.yaml

# 3. Create your own static pod
sudo tee /etc/kubernetes/manifests/lab-static.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lab-static
  namespace: kube-system
  labels:
    tier: control-plane
spec:
  containers:
  - name: busybox
    image: busybox:1.35
    command: ["sh", "-c", "while true; do echo 'Static pod running'; sleep 60; done"]
EOF

# 4. Exit and check from your Mac
exit
kubectl get pod lab-static-minikube -n kube-system
kubectl logs lab-static-minikube -n kube-system

# 5. Try to delete via kubectl (it will come back!)
kubectl delete pod lab-static-minikube -n kube-system
kubectl get pod -n kube-system    # it's back in seconds

# 6. Actually delete by removing the manifest
minikube ssh
sudo rm /etc/kubernetes/manifests/lab-static.yaml
exit

kubectl get pod lab-static-minikube -n kube-system
# Now it's gone
```

---

## Lab 03-C — Certificate Inspection

```bash
# 1. SSH into minikube
minikube ssh

# 2. Check certificate expiry
sudo kubeadm certs check-expiration

# 3. Inspect the apiserver certificate
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -E "Subject:|Issuer:|Not After"

# 4. Check the CA certificate
sudo openssl x509 -in /etc/kubernetes/pki/ca.crt \
  -noout -text | grep -E "Subject:|Not After"

# 5. Verify certificate chain
sudo openssl verify -CAfile /etc/kubernetes/pki/ca.crt \
  /etc/kubernetes/pki/apiserver.crt
# Should output: /etc/kubernetes/pki/apiserver.crt: OK

exit
```

---

## Checklist

- [ ] Can explain what kubeadm init does step by step
- [ ] Can join worker nodes using kubeadm join
- [ ] Can take and restore etcd snapshots (this IS on the exam)
- [ ] Know the cluster upgrade sequence (drain → upgrade kubeadm → apply → upgrade kubelet)
- [ ] Know what static pods are and where their manifests live
- [ ] Can inspect certificate expiry with kubeadm certs check-expiration

---

**Next:** [Module 04 — Kubernetes Core Objects](./Module-04-Kubernetes-Core-Objects.md)
