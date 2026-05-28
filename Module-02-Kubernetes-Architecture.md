# Module 02 — Kubernetes Architecture

> Understand how Kubernetes works internally before touching any workloads.

---

## 2.1 Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                            │
│                                                             │
│  ┌───────────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │ kube-apiserver│  │   etcd   │  │  kube-scheduler    │  │
│  │  (front door) │  │(database)│  │  (assigns pods)    │  │
│  └───────────────┘  └──────────┘  └────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            kube-controller-manager                   │  │
│  │  (node-controller, replication-controller, etc.)     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │ HTTPS (port 6443)
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    WORKER NODE                              │
│                                                             │
│  ┌───────────┐   ┌────────────┐   ┌───────────────────┐   │
│  │  kubelet  │   │ kube-proxy │   │ container runtime │   │
│  │(node agent│   │(networking)│   │(containerd/docker)│   │
│  └───────────┘   └────────────┘   └───────────────────┘   │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│  │  Pod 1   │ │  Pod 2   │ │  Pod 3   │                   │
│  └──────────┘ └──────────┘ └──────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.2 Control Plane Components

### kube-apiserver

- **The only component everything communicates with**
- Validates and processes all REST API calls
- Reads/writes to etcd
- Handles authentication, authorization, admission control
- Port: **6443** (HTTPS)

```bash
# View apiserver config (on cluster node)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check if apiserver is responding
kubectl cluster-info
curl -k https://localhost:6443/healthz
```

### etcd

- **Distributed key-value store — the cluster's brain/database**
- Stores ALL cluster state (objects, configs, secrets)
- Uses Raft consensus algorithm
- **Only backed up from etcd** — if etcd is lost, cluster is gone
- Default port: **2379** (client), **2380** (peer)

```bash
# Check etcd health (on master node)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# List all keys in etcd
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
```

### kube-scheduler

- Watches for Pods with no assigned node (`nodeName: ""`)
- Evaluates node filters (taints, affinity, resources)
- Assigns the best node by updating Pod spec
- Does NOT start the pod — kubelet does that

**Scheduling Algorithm:**
1. Filter: remove nodes that don't meet requirements
2. Score: rank remaining nodes
3. Bind: assign pod to highest-scoring node

### kube-controller-manager

- Runs all built-in controller loops in one process
- Each controller watches desired state vs actual state

| Controller | What it manages |
|-----------|----------------|
| Node controller | Detects node failures, marks NotReady |
| Replication controller | Maintains correct pod count |
| Endpoints controller | Populates Endpoints objects |
| Service Account controller | Creates default SAs |
| Job controller | Manages batch jobs |

```bash
# See controller-manager options
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

---

## 2.3 Worker Node Components

### kubelet

- **Agent running on every node**
- Communicates with kube-apiserver via HTTPS
- Reads PodSpecs, creates containers via container runtime
- Reports node and pod status back to apiserver
- Runs liveness/readiness probes
- Manages container lifecycle

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -n 50

# kubelet config file
cat /var/lib/kubelet/config.yaml

# kubelet kubeconfig
cat /etc/kubernetes/kubelet.conf
```

### kube-proxy

- Runs on every node as a DaemonSet
- Maintains network rules (iptables or IPVS)
- Implements the Service abstraction
- Routes traffic to correct pod IPs

```bash
# Check kube-proxy (it's a pod)
kubectl get pods -n kube-system | grep kube-proxy

# View kube-proxy logs
kubectl logs -n kube-system daemonset/kube-proxy
```

### Container Runtime

- Kubernetes uses **CRI (Container Runtime Interface)**
- Most clusters use **containerd** (Docker was deprecated in 1.24)
- Alternative: **CRI-O**

```bash
# Check which runtime is in use
kubectl get node -o wide    # shows container runtime column
kubectl describe node <name> | grep "Container Runtime"
```

---

## 2.4 Cluster Communication Flow

### Creating a Pod — Step by Step

```
kubectl apply -f pod.yaml
    │
    ▼
kube-apiserver  ← Validates YAML, authenticates, authorizes
    │             Stores pod object in etcd (status: Pending)
    ▼
kube-scheduler  ← Watches for unscheduled pods
    │             Selects best node, updates pod.spec.nodeName
    │             Writes back to etcd via apiserver
    ▼
kubelet (node)  ← Watches for pods assigned to its node
    │             Calls container runtime (containerd) to create containers
    │             Updates pod status back to apiserver → etcd
    ▼
Pod Running ✓
```

---

## 2.5 Kubernetes API

### API Groups

```
/api          → core API group (pods, services, namespaces...)
/apis         → named API groups
  /apps       → deployments, statefulsets, daemonsets
  /batch      → jobs, cronjobs
  /networking.k8s.io → ingress, networkpolicies
  /rbac.authorization.k8s.io → roles, rolebindings
  /storage.k8s.io → storageclasses, PVs
```

### API Versions

| Version | Meaning |
|---------|---------|
| v1 | Stable, production-ready |
| v1beta1 | Beta — mostly stable |
| v1alpha1 | Alpha — may change |

### API Resource Shortnames

```bash
kubectl api-resources                    # list all resources
kubectl api-resources --namespaced=true  # only namespaced resources
kubectl explain pod                      # describe a resource
kubectl explain pod.spec.containers      # nested field docs
kubectl explain deployment.spec.template
```

---

## Lab 02-A — Inspect Your Cluster

**Prerequisites:** Minikube running (`minikube start`)

```bash
# 1. Start minikube if not running
minikube start --cpus=2 --memory=4096

# 2. View cluster info
kubectl cluster-info

# 3. List all nodes
kubectl get nodes -o wide

# 4. Describe the node in detail
kubectl describe node minikube

# 5. Check component status (deprecated but still useful)
kubectl get componentstatuses

# 6. View all system pods (these ARE the control plane)
kubectl get pods -n kube-system

# 7. Check which pods make up the control plane
kubectl get pods -n kube-system -o wide | grep -E "apiserver|scheduler|controller|etcd"

# 8. Inspect the apiserver pod
kubectl describe pod kube-apiserver-minikube -n kube-system

# 9. Check kube-proxy
kubectl get daemonset -n kube-system kube-proxy
```

---

## Lab 02-B — Use kubectl explain

```bash
# Explore pod spec fields
kubectl explain pod
kubectl explain pod.metadata
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.containers.livenessProbe

# Explore deployment
kubectl explain deployment
kubectl explain deployment.spec.strategy

# Explore service
kubectl explain service.spec.type

# Get API version for a resource
kubectl explain deployment | head -5
# Output shows: VERSION: apps/v1

# List all API resources with versions
kubectl api-resources | head -30
```

---

## Lab 02-C — API Server Communication

```bash
# 1. Proxy the API server locally
kubectl proxy --port=8001 &

# 2. Now you can call the API directly with curl
curl http://localhost:8001/api/v1/namespaces
curl http://localhost:8001/api/v1/pods
curl http://localhost:8001/apis/apps/v1/deployments

# 3. Get a specific pod
kubectl run test-pod --image=nginx:alpine
# Wait for it to be running
kubectl get pod test-pod -o json | python3 -m json.tool | head -40

# 4. Direct API call
curl http://localhost:8001/api/v1/namespaces/default/pods/test-pod

# 5. Understand the apiserver URL structure
# http://localhost:8001/api/<version>/<resource>
# http://localhost:8001/apis/<group>/<version>/namespaces/<ns>/<resource>

# 6. Stop the proxy
kill %1

# 7. Cleanup
kubectl delete pod test-pod
```

---

## Lab 02-D — Trace Pod Creation (Deep Dive)

```bash
# Watch events as a pod is created — see the full flow
kubectl get events -w &

# Create a pod in another terminal
kubectl run trace-test --image=nginx:alpine

# You'll see events showing:
# 1. Scheduled — scheduler assigned it to a node
# 2. Pulling   — kubelet pulling the image
# 3. Pulled    — image downloaded
# 4. Created   — container created
# 5. Started   — container started

# Stop the watch
kill %1

# Check pod details
kubectl describe pod trace-test
# Look at the Events section at the bottom

# Cleanup
kubectl delete pod trace-test
```

---

## Lab 02-E — SSH into Minikube and Inspect Components

```bash
# SSH into the minikube node
minikube ssh

# Inside the node:

# Check static pod manifests (control plane lives here)
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml  etcd.yaml

# View apiserver manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check kubelet service
sudo systemctl status kubelet

# Check running containers (Kubernetes uses containerd)
sudo crictl ps

# List container images
sudo crictl images

# Check kubelet config
sudo cat /var/lib/kubelet/config.yaml

# Exit the node
exit
```

---

## Key Architecture Takeaways

| Component | Location | If it fails |
|-----------|----------|-------------|
| kube-apiserver | Control plane | Nothing works — all communication stops |
| etcd | Control plane | Cluster state lost — must restore backup |
| kube-scheduler | Control plane | New pods won't be scheduled (existing pods keep running) |
| controller-manager | Control plane | Controllers stop reconciling (replicas won't heal) |
| kubelet | Each worker node | Pods on that node stop being managed |
| kube-proxy | Each node | Service networking breaks on that node |

---

## Checklist

- [ ] Can explain what each control plane component does
- [ ] Know the port numbers: apiserver=6443, etcd=2379/2380
- [ ] Understand pod creation flow (api → scheduler → kubelet → runtime)
- [ ] Can use `kubectl explain` to look up any field
- [ ] Can inspect static pod manifests in `/etc/kubernetes/manifests/`
- [ ] Know the difference between kubelet and kube-proxy

---

**Next:** [Module 03 — Cluster Installation & Configuration](./Module-03-Cluster-Installation-Configuration.md)
