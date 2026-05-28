# Module 10 — Troubleshooting

> The highest-weight domain in modern CKA exams. Practice every scenario until it's muscle memory.

---

## Troubleshooting Mental Model

```
STEP 1: What's the symptom?
  - Pod not running
  - Service not reachable
  - Node NotReady
  - Cluster completely broken

STEP 2: kubectl get → find the broken object
STEP 3: kubectl describe → read Events section
STEP 4: kubectl logs → check application errors
STEP 5: journalctl → check system service errors
STEP 6: Fix → verify
```

---

## 10.1 Pod Troubleshooting

### Pod Phases

| Phase | Meaning |
|-------|---------|
| Pending | Not scheduled yet |
| Running | On a node, containers starting |
| Succeeded | All containers exited 0 |
| Failed | At least one container exited non-zero |
| Unknown | Node communication lost |

### Container States

| State | Meaning |
|-------|---------|
| Waiting | Not started (reason: ContainerCreating, ImagePullBackOff, etc.) |
| Running | Executing |
| Terminated | Exited (check exit code) |

### Common Pod Error Reasons

```bash
# CrashLoopBackOff — container keeps crashing
# Check:
kubectl logs pod-name --previous     # previous container logs
kubectl describe pod pod-name        # look at Events and Last State

# ImagePullBackOff / ErrImagePull
# Check:
kubectl describe pod pod-name        # wrong image name, registry access
# Fix:
kubectl set image pod/pod-name container=correct-image:tag
# Or fix the deployment manifest

# Pending (not scheduled)
kubectl describe pod pod-name        # Events: "0/N nodes available..."
# Causes:
# - No nodes match nodeSelector/affinity
# - All nodes tainted, no toleration
# - Insufficient CPU/memory
# - PVC not bound

# OOMKilled — out of memory
kubectl describe pod pod-name        # Last State: OOMKilled
# Fix: increase memory limits

# CreateContainerConfigError
kubectl describe pod pod-name        # missing ConfigMap or Secret
```

### Debugging Commands

```bash
# Full diagnosis sequence
kubectl get pod my-pod                    # phase
kubectl describe pod my-pod              # events + config
kubectl logs my-pod                       # app logs
kubectl logs my-pod --previous           # crashed container logs
kubectl logs my-pod -c sidecar           # specific container

# Run debug container in same pod (k8s 1.23+)
kubectl debug -it my-pod --image=busybox:1.35

# Run debug pod on the same node
kubectl debug node/worker1 -it --image=ubuntu

# Run temporary pod for network debugging
kubectl run debug --image=busybox:1.35 --rm -it -- sh
```

---

## 10.2 Node Troubleshooting

### Node NotReady Flow

```
kubectl get nodes → node is NotReady
    ↓
kubectl describe node <name> → check Conditions + Events
    ↓
SSH into node: ssh user@node-ip
    ↓
systemctl status kubelet       → is kubelet running?
journalctl -u kubelet -n 100   → kubelet error logs
    ↓
common issues:
- kubelet not running → systemctl restart kubelet
- certificate expired → kubeadm certs renew all
- disk pressure      → du -sh /var/lib/docker/*
- containerd down    → systemctl restart containerd
```

```bash
# Check kubelet on the node
systemctl status kubelet
systemctl is-active kubelet
journalctl -u kubelet -f                    # live logs
journalctl -u kubelet --since "5 min ago"   # recent logs

# Check containerd
systemctl status containerd
journalctl -u containerd -n 50

# Check if there's disk pressure
df -h
du -sh /var/lib/containerd/*

# Check memory
free -h

# Check if node can reach apiserver
curl -k https://<APISERVER_IP>:6443/healthz

# Restart services
systemctl restart kubelet
systemctl restart containerd
```

---

## 10.3 Control Plane Troubleshooting

### apiserver not responding

```bash
# Check if static pods are running
ls /etc/kubernetes/manifests/
# If kube-apiserver.yaml is missing → restore it

# Check apiserver pod
crictl ps | grep apiserver
crictl logs <container-id>

# Check apiserver process
ps aux | grep kube-apiserver

# Check apiserver port
ss -tlnp | grep 6443
curl -k https://localhost:6443/healthz

# Common causes:
# 1. Wrong certificate (inspect manifest)
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Look for --cert-dir and --etcd-* flags

# 2. etcd is down
systemctl status etcd   # if external etcd
crictl ps | grep etcd   # if static pod etcd
```

### scheduler / controller-manager down

```bash
# These are static pods — check manifests
ls /etc/kubernetes/manifests/
crictl ps | grep scheduler
crictl ps | grep controller

# If pods are missing, check logs
crictl logs <container-id>

# Common fix: incorrect image tag in manifest
sudo vim /etc/kubernetes/manifests/kube-scheduler.yaml
```

### etcd recovery

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# If etcd is corrupted, restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-new

# Update etcd manifest to use new data dir
sudo vim /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd → --data-dir=/var/lib/etcd-new
# Change: hostPath.path: /var/lib/etcd → /var/lib/etcd-new

# Wait for etcd to restart
watch crictl ps | grep etcd
```

---

## 10.4 DNS Troubleshooting

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from inside a pod
kubectl run dns-test --image=busybox:1.35 --rm -it -- sh
  nslookup kubernetes.default
  nslookup my-service.my-namespace.svc.cluster.local
  cat /etc/resolv.conf
  exit

# Check CoreDNS ConfigMap
kubectl get cm coredns -n kube-system -o yaml

# Check if kube-dns service exists
kubectl get svc -n kube-system kube-dns

# Common issues:
# 1. CoreDNS pods not running → kubectl describe + fix image/config
# 2. CoreDNS ConfigMap wrong → kubectl edit cm coredns -n kube-system
# 3. kube-dns service missing → recreate it
# 4. Pod's /etc/resolv.conf wrong → check kubelet --cluster-dns flag
```

---

## 10.5 Networking Troubleshooting

```bash
# Check if service exists and has endpoints
kubectl get svc my-service
kubectl get endpoints my-service
# If endpoints are empty → label selector mismatch

# Verify pod labels match service selector
kubectl get pod -l app=web --show-labels
kubectl describe svc my-service | grep Selector

# Test connectivity from a pod
kubectl run nettest --image=busybox:1.35 --rm -it -- sh
  wget -qO- http://my-service:80
  wget -qO- http://my-service.my-namespace.svc.cluster.local:80
  ping my-service
  exit

# Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check iptables rules (on node)
sudo iptables -t nat -L | grep my-service

# Check NetworkPolicy blocking traffic
kubectl get networkpolicy -n my-namespace
kubectl describe networkpolicy my-policy -n my-namespace

# Port connectivity test
kubectl run porttest --image=busybox:1.35 --rm -it -- \
  nc -zv my-service 80
```

---

## 10.6 Storage Troubleshooting

```bash
# PVC Pending
kubectl describe pvc my-pvc
# Look for: "no persistent volumes available" or "volume node affinity conflict"

# Common PVC issues:
# 1. No matching PV → storageClassName mismatch, size too large, wrong accessMode
# 2. StorageClass provisioner not available
# 3. Node affinity conflict (PV bound to different node)

kubectl get pv                          # check PV status and storageclassname
kubectl describe pv my-pv              # check access modes, capacity

# Pod with volume mount issue
kubectl describe pod my-pod
# Events: "Unable to attach volume", "FailedMount"

# Force PVC deletion (if stuck in Terminating)
kubectl patch pvc my-pvc \
  -p '{"metadata":{"finalizers":null}}'
```

---

## 10.7 Certificate Expiration

```bash
# Check cert expiry
sudo kubeadm certs check-expiration

# Renew all certs
sudo kubeadm certs renew all

# After renewing, restart control plane pods
sudo mv /etc/kubernetes/manifests /tmp/manifests-bak
sleep 5
sudo mv /tmp/manifests-bak /etc/kubernetes/manifests

# Or kill the pods (they auto-restart as static pods)
sudo crictl rm $(sudo crictl ps -a | grep kube-apiserver | awk '{print $1}')

# Update kubeconfig after renewal
sudo cat /etc/kubernetes/admin.conf    # copy this to ~/.kube/config
```

---

## Lab 10-A — Fix a Broken Pod

```bash
# Create broken pods to fix

# SCENARIO 1: Wrong image
kubectl run broken1 --image=nginx:DOESNOTEXIST999
kubectl get pod broken1               # ErrImagePull or ImagePullBackOff
kubectl describe pod broken1
# Fix:
kubectl set image pod/broken1 broken1=nginx:alpine
kubectl get pod broken1               # should be Running

# SCENARIO 2: CrashLoopBackOff
cat > /tmp/crashloop.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo Starting; exit 1"]   # exits with error
EOF

kubectl apply -f /tmp/crashloop.yaml
kubectl get pod crashloop -w    # watch it crash and restart

kubectl describe pod crashloop
kubectl logs crashloop --previous   # see what happened

# Fix: update command to not fail
kubectl patch pod crashloop -p \
  '{"spec":{"containers":[{"name":"app","command":["sh","-c","echo Running; sleep 3600"]}]}}'
# Note: you cannot patch most pod spec fields without recreating
# In exam: delete and recreate with fixed YAML
kubectl delete pod crashloop
cat > /tmp/crashloop-fixed.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo Running; sleep 3600"]
EOF
kubectl apply -f /tmp/crashloop-fixed.yaml
kubectl get pod crashloop   # Running

# SCENARIO 3: Missing ConfigMap
cat > /tmp/missing-cm.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: needs-config
spec:
  containers:
  - name: app
    image: nginx:alpine
    envFrom:
    - configMapRef:
        name: app-settings   # this ConfigMap doesn't exist!
EOF

kubectl apply -f /tmp/missing-cm.yaml
kubectl describe pod needs-config
# Events: Error: configmap "app-settings" not found

# Fix: create the missing ConfigMap
kubectl create configmap app-settings \
  --from-literal=LOG_LEVEL=info \
  --from-literal=APP_PORT=8080

# Pod will pick it up after recreation
kubectl delete pod needs-config
kubectl apply -f /tmp/missing-cm.yaml
kubectl get pod needs-config   # Running

kubectl delete pod broken1 crashloop needs-config
```

---

## Lab 10-B — Fix a Broken Service

```bash
# 1. Create a deployment and broken service
kubectl create deployment webapp --image=nginx:alpine --replicas=3
kubectl expose deployment webapp --port=80 --target-port=80

# Verify it works
kubectl run tester --image=busybox:1.35 --rm -it -- wget -qO- http://webapp
# Should succeed

# 2. Break the service (wrong selector)
kubectl patch svc webapp -p '{"spec":{"selector":{"app":"wrong-label"}}}'

# 3. Debug
kubectl get endpoints webapp    # EMPTY — no endpoints!
kubectl describe svc webapp     # shows selector: app=wrong-label

# 4. Fix the selector
kubectl patch svc webapp -p '{"spec":{"selector":{"app":"webapp"}}}'

# 5. Verify
kubectl get endpoints webapp   # should show pod IPs
kubectl run tester --image=busybox:1.35 --rm -it -- wget -qO- http://webapp

kubectl delete deployment webapp
kubectl delete svc webapp
```

---

## Lab 10-C — Node Troubleshooting on Minikube

```bash
# 1. Simulate kubelet failure
minikube ssh

# Stop kubelet
sudo systemctl stop kubelet
exit

# 2. On your Mac — watch node status
kubectl get nodes -w &
WATCH_PID=$!
sleep 40    # node will go NotReady after ~40s
kill $WATCH_PID

kubectl get nodes    # should show NotReady

# 3. Describe the node
kubectl describe node minikube | grep -A10 "Conditions:"
# Shows: Ready=False, Reason=KubeletNotReady or NodeStatusUnknown

# 4. Fix: restart kubelet
minikube ssh
sudo systemctl start kubelet
sudo systemctl status kubelet    # verify it's active
exit

# 5. Wait for node to become Ready
kubectl get nodes -w    # should go back to Ready in ~30s
```

---

## Lab 10-D — DNS Failure Fix

```bash
# 1. Break CoreDNS
kubectl scale deployment coredns --replicas=0 -n kube-system

# 2. Test DNS fails
kubectl run dns-test --image=busybox:1.35 --rm -it -- \
  nslookup kubernetes.default
# Should fail / timeout

# 3. Check what's wrong
kubectl get pods -n kube-system | grep coredns
# No pods!

kubectl get deployment -n kube-system coredns
# READY: 0/0

# 4. Fix: restore CoreDNS
kubectl scale deployment coredns --replicas=2 -n kube-system

kubectl get pods -n kube-system -l k8s-app=kube-dns -w
# Wait for Running

# 5. Verify DNS works again
kubectl run dns-test --image=busybox:1.35 --rm -it -- \
  nslookup kubernetes.default
# Should return answer
```

---

## 10.8 Troubleshooting Cheatsheet

```bash
# ========== PODS ==========
kubectl get pods -A                              # all pods, all namespaces
kubectl get pods -A --field-selector status.phase!=Running
kubectl describe pod <name>                      # full detail + events
kubectl logs <name> -p                           # previous container
kubectl exec -it <name> -- sh                    # shell into pod

# ========== NODES ==========
kubectl get nodes -o wide
kubectl describe node <name>
kubectl cordon <node>       # prevent new scheduling
kubectl drain <node> --ignore-daemonsets   # evict pods
kubectl uncordon <node>     # re-enable scheduling

# ========== SERVICES ==========
kubectl get endpoints <svc>                      # check pod IPs
kubectl describe svc <svc>                       # check selector

# ========== CLUSTER ==========
kubectl cluster-info
kubectl get events -A --sort-by=.lastTimestamp

# ========== ON NODE ==========
systemctl status kubelet
journalctl -u kubelet -n 50 --no-pager
crictl ps                                        # running containers
crictl logs <container-id>
ls /etc/kubernetes/manifests/                    # static pod manifests
```

---

## Scenario Quick Reference

| Symptom | First Check | Common Fix |
|---------|-------------|-----------|
| Pod: ImagePullBackOff | `kubectl describe pod` | Fix image name/tag |
| Pod: CrashLoopBackOff | `kubectl logs --previous` | Fix app crash |
| Pod: Pending | `kubectl describe pod` → Events | Fix affinity/taints/resources/PVC |
| Pod: OOMKilled | `kubectl describe pod` → Last State | Increase memory limits |
| Service: No endpoints | `kubectl get endpoints` | Fix label selector |
| Service: Can't reach | Test from debug pod | Check NetworkPolicy |
| Node: NotReady | `journalctl -u kubelet` | Restart kubelet/containerd |
| DNS fails | `kubectl get pods -n kube-system` | Restore CoreDNS |
| Cluster broken | Check manifests in `/etc/kubernetes/manifests/` | Fix manifest or restore etcd |
| Cert expired | `kubeadm certs check-expiration` | `kubeadm certs renew all` |

---

## Checklist

- [ ] Can diagnose and fix ImagePullBackOff
- [ ] Can diagnose and fix CrashLoopBackOff (logs --previous)
- [ ] Can diagnose and fix Pending pods (affinity, taints, resources)
- [ ] Can diagnose and fix OOMKilled pods
- [ ] Can diagnose broken services (empty endpoints → label selector)
- [ ] Can troubleshoot NotReady nodes (kubelet logs)
- [ ] Can restore CoreDNS when broken
- [ ] Can diagnose control plane failures from static pod manifests
- [ ] Know the etcd restore procedure

---

**Next:** [Module 11 — Helm & Package Management](./Module-11-Helm-Package-Management.md)
