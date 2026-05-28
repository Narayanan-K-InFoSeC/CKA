# Module 09 — Logging & Monitoring

> Knowing how to extract logs and interpret events is essential for troubleshooting (Module 10).

---

## 9.1 Container & Pod Logs

```bash
# Basic log commands
kubectl logs <pod-name>
kubectl logs <pod-name> -n <namespace>

# Follow (stream) logs in real time
kubectl logs <pod-name> -f

# Last N lines
kubectl logs <pod-name> --tail=50

# Since a time period
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=5m

# With timestamps
kubectl logs <pod-name> --timestamps=true

# Specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# All containers in a pod
kubectl logs <pod-name> --all-containers=true

# Previous container (after a crash/restart)
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -p

# Logs from all pods with a label
kubectl logs -l app=web --all-containers=true

# Logs from a deployment (all pods)
kubectl logs deployment/my-app
kubectl logs deployment/my-app -f
```

---

## 9.2 Viewing Events

Events are the most useful quick-debugging tool.

```bash
# All events in default namespace
kubectl get events

# Events in a namespace sorted by time
kubectl get events -n kube-system --sort-by='.lastTimestamp'

# Watch events in real time
kubectl get events -w

# Events for a specific resource
kubectl describe pod my-pod     # Events section at bottom
kubectl describe node worker1   # node events
kubectl describe deployment web

# Only warning events
kubectl get events --field-selector type=Warning

# Events for a specific pod name
kubectl get events --field-selector involvedObject.name=my-pod

# Events in all namespaces
kubectl get events -A
```

---

## 9.3 Metrics Server

Provides CPU and memory usage for pods and nodes.

```bash
# Install on minikube
minikube addons enable metrics-server

# Wait for it to start
kubectl get pods -n kube-system | grep metrics

# Top nodes (resource usage per node)
kubectl top nodes

# Top pods in all namespaces
kubectl top pods -A

# Top pods with containers
kubectl top pods --containers=true

# Top pods sorted by CPU
kubectl top pods --sort-by=cpu

# Top pods sorted by memory
kubectl top pods --sort-by=memory
```

---

## 9.4 Node Health Inspection

```bash
# Quick node status
kubectl get nodes
kubectl get nodes -o wide

# Detailed node info
kubectl describe node <node-name>

# Important sections in describe node:
# - Conditions: Ready, MemoryPressure, DiskPressure, PIDPressure
# - Capacity vs Allocatable (actual available after system reservation)
# - Non-terminated Pods (what's running)
# - Events

# Check node conditions programmatically
kubectl get node minikube \
  -o jsonpath='{.status.conditions[*].type}'

kubectl get nodes -o custom-columns=\
  "NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason"
```

### Node Conditions

| Condition | Healthy Value | Issue |
|-----------|--------------|-------|
| Ready | True | False means node is not ready |
| MemoryPressure | False | True means low memory |
| DiskPressure | False | True means low disk |
| PIDPressure | False | True means too many processes |
| NetworkUnavailable | False | True means no network |

---

## 9.5 Cluster Monitoring Basics

```bash
# Check all resources across cluster
kubectl get all -A

# Check resource utilization
kubectl top nodes
kubectl top pods -A

# Check pod restart counts
kubectl get pods -A | awk '$4 > 0'    # pods with restarts

# Check failed pods
kubectl get pods -A --field-selector status.phase=Failed

# Check pending pods
kubectl get pods -A --field-selector status.phase=Pending

# Check all events sorted by time (full cluster view)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
```

---

## 9.6 Monitoring Architecture (Conceptual)

```
Pods → metrics-server → kubectl top (short-term)

Pods → Prometheus → Grafana (long-term, production)

Node → node-exporter → Prometheus → Grafana

Kubernetes API → kube-state-metrics → Prometheus → Grafana
```

For the CKA exam, only `metrics-server` and `kubectl top` are needed.

---

## Lab 09-A — Log Analysis

```bash
# 1. Create a pod that generates various log types
cat > /tmp/logger-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: logger
spec:
  containers:
  - name: app
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "INFO: Application starting"
      echo "INFO: Connecting to database"
      echo "WARNING: Database response slow"
      echo "ERROR: Failed to connect to cache"
      echo "INFO: Using fallback mode"
      for i in $(seq 1 20); do
        echo "INFO: Processing request $i at $(date)"
        sleep 2
      done
EOF

kubectl apply -f /tmp/logger-pod.yaml

# 2. Wait for pod to produce logs
sleep 5

# 3. Practice log commands
kubectl logs logger                        # all logs
kubectl logs logger --tail=5               # last 5 lines
kubectl logs logger -f &                   # follow in background
LOG_PID=$!
sleep 6
kill $LOG_PID

kubectl logs logger --since=10s           # last 10 seconds
kubectl logs logger --timestamps=true      # with timestamps

# 4. Filter logs with grep
kubectl logs logger | grep "ERROR"
kubectl logs logger | grep -E "ERROR|WARNING"
kubectl logs logger | grep -v "INFO"       # exclude INFO

# 5. Count log lines
kubectl logs logger | wc -l

kubectl delete pod logger
```

---

## Lab 09-B — Events & Debugging

```bash
# 1. Watch events while creating a pod
kubectl get events -w &
EVENTS_PID=$!

# 2. Create a pod that fails (wrong image)
kubectl run failing-pod --image=nginx:nonexistent-version-999

# 3. Watch events — you'll see ErrImagePull
sleep 10
kill $EVENTS_PID

# 4. Inspect the pod
kubectl describe pod failing-pod
# Events section shows: Failed to pull image...

# 5. Check events for just this pod
kubectl get events --field-selector involvedObject.name=failing-pod

# 6. Create a pod with memory issue
cat > /tmp/oom-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: mem-eater
    image: polinux/stress
    resources:
      limits:
        memory: "32Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M"]
EOF

kubectl apply -f /tmp/oom-pod.yaml
sleep 10
kubectl describe pod oom-pod | grep -A5 "Last State:"
# Shows: Reason: OOMKilled
kubectl get events --field-selector involvedObject.name=oom-pod

kubectl delete pod failing-pod oom-pod
```

---

## Lab 09-C — Metrics Server

```bash
# 1. Enable metrics server
minikube addons enable metrics-server

# 2. Wait for metrics to be available (takes ~60 seconds)
kubectl get pods -n kube-system -l k8s-app=metrics-server -w

# 3. Create some workloads
kubectl create deployment cpu-burn --image=containerstack/cpustress \
  --replicas=2
kubectl create deployment memory-use --image=nginx:alpine \
  --replicas=3

# Wait for pods
kubectl get pods -w &
sleep 20
kill %1

# 4. Check resource usage
kubectl top nodes

kubectl top pods
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
kubectl top pods --containers=true   # per-container view

# 5. Check specific namespace
kubectl top pods -n kube-system

# 6. Identify resource-hungry pods
kubectl top pods -A --sort-by=memory | head -10

kubectl delete deployment cpu-burn memory-use
```

---

## Log Locations on Nodes (for Node Troubleshooting)

```bash
# SSH into minikube node
minikube ssh

# Container logs (managed by container runtime)
sudo ls /var/log/containers/
sudo cat /var/log/containers/kube-apiserver-minikube_kube-system_*.log | tail -20

# Pod logs
sudo ls /var/log/pods/

# System logs
sudo journalctl -u kubelet -n 50
sudo journalctl -u containerd -n 50

# Kernel logs (for node issues)
sudo dmesg | tail -20

# System journal
sudo journalctl --since "10 min ago"
exit
```

---

## Checklist

- [ ] Know all `kubectl logs` flags: `-f`, `--tail`, `--since`, `-p`, `-c`
- [ ] Can filter logs with `grep` and count with `wc`
- [ ] Can use `kubectl get events` to debug issues
- [ ] Can sort events by timestamp
- [ ] Know how to enable and use metrics-server
- [ ] Can use `kubectl top nodes` and `kubectl top pods`
- [ ] Know the five node conditions (Ready, MemoryPressure, DiskPressure, etc.)

---

**Next:** [Module 10 — Troubleshooting](./Module-10-Troubleshooting.md)
