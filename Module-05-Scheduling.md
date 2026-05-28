# Module 05 — Scheduling

> Control where pods land and how resources are distributed across nodes.

---

## 5.1 Node Selectors (Simple)

Direct pod to nodes with a specific label.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"          # node must have this label
  containers:
  - name: ml-workload
    image: tensorflow/tensorflow:latest-gpu
```

```bash
# Label a node
kubectl label node worker1 gpu=true

# Verify label
kubectl get node worker1 --show-labels

# Remove label
kubectl label node worker1 gpu-
```

---

## 5.2 Node Affinity (Advanced Selectors)

More expressive than nodeSelector — supports operators and soft rules.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      # HARD rule — pod won't schedule if not met
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
          - key: node-type
            operator: NotIn
            values:
            - spot

      # SOFT rule — prefer but not required
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80           # 1-100, higher = more preferred
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
      - weight: 20
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx:alpine
```

### Operators

| Operator | Meaning |
|----------|---------|
| In | label value in list |
| NotIn | label value not in list |
| Exists | label key exists |
| DoesNotExist | label key absent |
| Gt | numeric value greater than |
| Lt | numeric value less than |

---

## 5.3 Pod Affinity & Anti-Affinity

Schedule pods relative to other pods (not nodes).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  affinity:
    # Run on same node as a pod with app=cache (co-location)
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname   # same node

    # Don't run on same node as another web pod (spread)
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
  containers:
  - name: web
    image: nginx:alpine
```

---

## 5.4 Taints & Tolerations

**Taint** = repel pods from a node (applied to nodes)
**Toleration** = allow a pod to land on a tainted node (applied to pods)

### Taint Effects

| Effect | Behavior |
|--------|---------|
| NoSchedule | New pods won't be scheduled (existing stay) |
| PreferNoSchedule | Soft — try to avoid but not guaranteed |
| NoExecute | New pods rejected + existing pods evicted if no toleration |

```bash
# Taint a node
kubectl taint node worker1 key=value:NoSchedule
kubectl taint node worker1 env=prod:NoExecute
kubectl taint node worker1 dedicated=gpu:NoSchedule

# List taints
kubectl describe node worker1 | grep Taint

# Remove taint (append -)
kubectl taint node worker1 key=value:NoSchedule-
kubectl taint node worker1 dedicated-       # remove all taints for key

# Control plane has a default taint:
# node-role.kubernetes.io/control-plane:NoSchedule
# That's why regular pods don't land on master
```

### Toleration in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerated-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoExecute"
    tolerationSeconds: 3600    # evict after 1 hour even with toleration

  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300     # tolerate not-ready for 5 min (default behavior)

  containers:
  - name: app
    image: nginx:alpine
```

```yaml
# Tolerate ALL taints (wildcard)
tolerations:
- operator: "Exists"
```

---

## 5.5 Resource Requests & Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:          # minimum guaranteed
        cpu: "250m"      # 250 millicores = 0.25 CPU
        memory: "64Mi"   # 64 MiB
      limits:            # maximum allowed
        cpu: "500m"
        memory: "128Mi"
```

### CPU Units

```
1 CPU  = 1000m (millicores) = 1 vCPU = 1 core
500m   = 0.5 CPU
100m   = 0.1 CPU (10% of one core)
```

### Memory Units

```
Ki = Kibibytes (1024 bytes)
Mi = Mebibytes (1024 Ki)
Gi = Gibibytes
K  = Kilobytes (1000 bytes) — different!
M  = Megabytes
G  = Gigabytes
```

---

## 5.6 LimitRange

Set default resource requests/limits per namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:             # default limit if not set
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:      # default request if not set
      cpu: "100m"
      memory: "64Mi"
    max:                 # maximum allowed per container
      cpu: "2"
      memory: "1Gi"
    min:                 # minimum allowed
      cpu: "50m"
      memory: "32Mi"
```

---

## 5.7 ResourceQuota

Limit total resource consumption per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    persistentvolumeclaims: "5"
    services: "5"
```

---

## 5.8 QoS Classes

Kubernetes assigns QoS class based on resources:

| Class | Condition | Eviction priority |
|-------|-----------|------------------|
| **Guaranteed** | limits == requests (both set) | Last to evict |
| **Burstable** | requests < limits | Middle |
| **BestEffort** | no requests/limits set | First to evict |

```bash
kubectl describe pod my-pod | grep "QoS Class"
```

---

## 5.9 PriorityClass

Higher priority pods evict lower priority ones if resources are scarce.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production workloads"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:alpine
```

---

## 5.10 Manual Scheduling (nodeName)

Bypass the scheduler entirely — assign pod directly to a node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: worker1      # skip scheduler
  containers:
  - name: app
    image: nginx:alpine
```

---

## Lab 05-A — Node Selectors and Affinity

```bash
# 1. Start minikube with multiple nodes
minikube start --nodes=2 --cpus=2 --memory=2048

# 2. List nodes
kubectl get nodes

# 3. Label nodes
kubectl label node minikube disk=ssd
kubectl label node minikube-m02 disk=hdd

# Verify labels
kubectl get nodes --show-labels

# 4. Create pod with nodeSelector
cat > /tmp/nodeselector-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disk: ssd
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl apply -f /tmp/nodeselector-pod.yaml

# 5. Verify it landed on correct node
kubectl get pod ssd-pod -o wide   # should be on minikube (ssd node)

# 6. Create pod with node affinity
cat > /tmp/affinity-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
            - nvme
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - minikube
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl apply -f /tmp/affinity-pod.yaml
kubectl get pod affinity-pod -o wide

# 7. Test: pod with unmatched affinity stays Pending
cat > /tmp/pending-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - nonexistent-disk-type
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl apply -f /tmp/pending-pod.yaml
kubectl get pod pending-pod     # should be Pending
kubectl describe pod pending-pod | grep -A5 "Events:"
# Will show: 0/2 nodes are available: node(s) didn't match node affinity

kubectl delete pod ssd-pod affinity-pod pending-pod
```

---

## Lab 05-B — Taints & Tolerations

```bash
# 1. Taint the second node
kubectl taint node minikube-m02 team=backend:NoSchedule

# 2. Describe to verify taint
kubectl describe node minikube-m02 | grep -i taint

# 3. Create pod without toleration — should NOT land on minikube-m02
kubectl run no-toleration --image=nginx:alpine
kubectl get pod no-toleration -o wide    # lands on minikube

# 4. Create pod WITH toleration — can land anywhere
cat > /tmp/toleration-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl apply -f /tmp/toleration-pod.yaml
kubectl get pod with-toleration -o wide    # can land on minikube-m02

# 5. Test NoExecute effect (evicts existing pods)
kubectl run running-pod --image=nginx:alpine
kubectl get pod running-pod -o wide   # note which node

# Taint that node with NoExecute
NODE=$(kubectl get pod running-pod -o jsonpath='{.spec.nodeName}')
kubectl taint node $NODE evict-test=true:NoExecute

# Pod should be evicted
kubectl get pod running-pod   # should be gone or on other node

# 6. Remove taints
kubectl taint node minikube-m02 team-
kubectl taint node $NODE evict-test-

kubectl delete pod no-toleration with-toleration
```

---

## Lab 05-C — Resource Requests, Limits and QoS

```bash
# 1. Create pods with different QoS classes
cat > /tmp/qos-demo.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"    # same as request = Guaranteed
        memory: "64Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:          # higher than request = Burstable
        cpu: "200m"
        memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: app
    image: nginx:alpine
    # no resources = BestEffort
EOF

kubectl apply -f /tmp/qos-demo.yaml

# 2. Check QoS classes
kubectl describe pod qos-guaranteed | grep "QoS Class"
kubectl describe pod qos-burstable  | grep "QoS Class"
kubectl describe pod qos-besteffort | grep "QoS Class"

# 3. Try exceeding memory limit (limits enforced by kernel)
cat > /tmp/oom-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    resources:
      limits:
        memory: "64Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "128M", "--vm-hang", "1"]
EOF

kubectl apply -f /tmp/oom-pod.yaml
kubectl get pod oom-test -w    # watch for OOMKilled

kubectl describe pod oom-test | grep -A5 "Last State:"
# Shows: Reason: OOMKilled

kubectl delete pod qos-guaranteed qos-burstable qos-besteffort oom-test
```

---

## Checklist

- [ ] Can use `nodeSelector` to target specific nodes
- [ ] Can write nodeAffinity with `required` and `preferred` rules
- [ ] Understand taints (node side) vs tolerations (pod side)
- [ ] Know all three taint effects: NoSchedule, PreferNoSchedule, NoExecute
- [ ] Understand CPU millicores and memory units (Mi vs M)
- [ ] Know the three QoS classes and what determines each
- [ ] Can use `kubectl taint` to add and remove taints

---

**Next:** [Module 06 — Services & Networking](./Module-06-Services-Networking.md)
