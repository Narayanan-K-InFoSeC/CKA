# Module 12 — Exam Preparation

> Speed, accuracy, and knowing where to look in the docs. This module is pure practice strategy.

---

## 12.1 Exam Environment

- **Browser-based terminal** (no local tools)
- **1 allowed tab:** kubernetes.io/docs (and subdomains)
- **2 hours, ~17 tasks**
- **Each task has a weight (2-8%)**
- Always read the full question — it tells you which cluster and namespace to use

### Pre-Exam Setup (first 60 seconds)

```bash
# Set up aliases — these save minutes over 2 hours
alias k=kubectl
alias kn='kubectl -n'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'

# Enable kubectl autocomplete
source <(kubectl completion bash)    # if using bash
source <(kubectl completion zsh)     # if using zsh

# Set default editor to nano (faster than vim for most)
export KUBE_EDITOR=nano

# Or if you're comfortable with vim:
export KUBE_EDITOR=vim
```

---

## 12.2 kubectl Shortcuts That Matter

### Dry-run + output YAML (fastest way to generate templates)

```bash
# Pod
kubectl run nginx --image=nginx:alpine --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment web --image=nginx:alpine --replicas=3 \
  --dry-run=client -o yaml > deploy.yaml

# Service
kubectl expose deployment web --port=80 --type=NodePort \
  --dry-run=client -o yaml > svc.yaml

# ConfigMap
kubectl create configmap myconfig --from-literal=key=val \
  --dry-run=client -o yaml

# Secret
kubectl create secret generic mysecret --from-literal=pass=secret \
  --dry-run=client -o yaml

# ServiceAccount
kubectl create serviceaccount mysa --dry-run=client -o yaml

# Role
kubectl create role myrole --verb=get,list --resource=pods \
  --dry-run=client -o yaml

# RoleBinding
kubectl create rolebinding myrb --role=myrole --user=alice \
  --dry-run=client -o yaml

# Job
kubectl create job myjob --image=busybox -- echo hello \
  --dry-run=client -o yaml

# CronJob
kubectl create cronjob mycron --image=busybox --schedule="*/5 * * * *" \
  -- echo hello --dry-run=client -o yaml
```

### Useful Output Formats

```bash
# JSON path — extract specific fields
kubectl get pod nginx -o jsonpath='{.spec.nodeName}'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Custom columns
kubectl get pods -o custom-columns="NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase"
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory"

# Wide output
kubectl get pods -o wide
kubectl get nodes -o wide

# YAML of running resource (export)
kubectl get pod nginx -o yaml > nginx-backup.yaml
kubectl get deployment web -o yaml > web-deploy.yaml
```

### Namespace Shortcuts

```bash
# Always specify -n or set the namespace context
kubectl get pods -n kube-system
kubectl config set-context --current --namespace=production

# All namespaces
kubectl get pods -A
kubectl get pods --all-namespaces
```

---

## 12.3 Imperative Commands Reference

These are faster than writing YAML during the exam.

```bash
# ============ WORKLOADS ============
kubectl run pod1 --image=nginx:alpine
kubectl run pod1 --image=nginx:alpine --labels="app=web,env=prod"
kubectl run pod1 --image=nginx:alpine --env="PORT=8080"
kubectl run pod1 --image=nginx:alpine --port=80
kubectl run pod1 --image=nginx:alpine --command -- /bin/sh -c "sleep 3600"

kubectl create deployment web --image=nginx:alpine
kubectl create deployment web --image=nginx:alpine --replicas=3
kubectl scale deployment web --replicas=5

# ============ SERVICES ============
kubectl expose pod pod1 --port=80 --name=pod1-svc
kubectl expose deployment web --port=80 --type=NodePort
kubectl expose deployment web --port=80 --target-port=8080 --type=ClusterIP

# ============ CONFIG ============
kubectl create configmap myconfig --from-literal=key=value
kubectl create configmap myconfig --from-file=app.conf
kubectl create secret generic mysecret --from-literal=password=secret
kubectl create secret tls mytls --cert=cert.crt --key=cert.key

# ============ RBAC ============
kubectl create serviceaccount mysa
kubectl create role myrole --verb=get,list,watch --resource=pods
kubectl create clusterrole myclusterrole --verb=get,list --resource=nodes
kubectl create rolebinding myrb --role=myrole --user=alice
kubectl create rolebinding myrb --role=myrole --serviceaccount=default:mysa
kubectl create clusterrolebinding mycrb --clusterrole=myclusterrole --user=bob

# ============ NAMESPACE ============
kubectl create namespace dev
kubectl delete namespace dev

# ============ LABELS ============
kubectl label pod pod1 env=prod
kubectl label node worker1 disk=ssd
kubectl annotate pod pod1 description="test pod"

# ============ TAINT ============
kubectl taint node worker1 key=value:NoSchedule
kubectl taint node worker1 key=value:NoSchedule-   # remove
```

---

## 12.4 Time Management Strategy

### Task Prioritization

```
High-weight tasks (6-8%):
→ Do these FIRST (even if harder)
→ Worth 1-2 extra tasks in time investment

Medium-weight tasks (4-5%):
→ Do these second

Low-weight tasks (2-3%):
→ Quick wins — do them fast or last

SKIP and FLAG:
→ If a task takes > 8 minutes, FLAG it and move on
→ Come back with remaining time
```

### Per-Task Time Budget

```
Total: 120 minutes for ~17 tasks
Average: ~7 minutes per task

Easy tasks: 3-4 minutes
Medium tasks: 6-8 minutes
Hard tasks: 10-12 minutes (2 max in exam)
```

---

## 12.5 Using Kubernetes Documentation

### Best Pages to Bookmark

```
kubernetes.io/docs/concepts/workloads/pods/
kubernetes.io/docs/concepts/storage/persistent-volumes/
kubernetes.io/docs/concepts/services-networking/service/
kubernetes.io/docs/concepts/services-networking/network-policies/
kubernetes.io/docs/concepts/security/rbac-good-practices/
kubernetes.io/docs/reference/kubectl/cheatsheet/
kubernetes.io/docs/tasks/administer-cluster/kubeadm/
kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
```

### Search Techniques

```
1. Use Ctrl+F on the docs page — fastest
2. Google: "site:kubernetes.io/docs networkpolicy"
3. Know the URL patterns:
   /docs/concepts/ → theory
   /docs/tasks/    → how-to guides
   /docs/reference/ → API reference
```

---

## 12.6 Common Exam Scenarios

### Scenario: Create a pod in a specific namespace with a service account

```bash
kubectl run mypod \
  --image=nginx:alpine \
  --serviceaccount=mysvc \
  -n target-namespace \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Scenario: Expose a deployment and verify connectivity

```bash
kubectl expose deployment myapp --port=80 --type=ClusterIP
kubectl run test --image=busybox --rm -it -- wget -qO- http://myapp
```

### Scenario: Scale and roll back a deployment

```bash
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=nginx:1.25
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp
```

### Scenario: Create RBAC for a service account

```bash
kubectl create serviceaccount ci-bot -n ci
kubectl create role ci-role --verb=get,list,create --resource=pods -n ci
kubectl create rolebinding ci-rb --role=ci-role --serviceaccount=ci:ci-bot -n ci
kubectl auth can-i create pods --as=system:serviceaccount:ci:ci-bot -n ci
```

### Scenario: Create persistent storage

```bash
# 1. Create PV
cat > pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
  storageClassName: manual
EOF
kubectl apply -f pv.yaml

# 2. Create PVC
cat > pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
EOF
kubectl apply -f pvc.yaml

# 3. Use in pod
kubectl run storage-pod --image=nginx:alpine \
  --dry-run=client -o yaml > pod.yaml
# Edit pod.yaml to add volume and volumeMount, then apply
```

### Scenario: Drain a node

```bash
kubectl drain worker1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Do the maintenance

kubectl uncordon worker1
```

### Scenario: Fix a failing pod (CrashLoopBackOff)

```bash
kubectl describe pod failing-pod         # check events
kubectl logs failing-pod --previous      # previous logs
kubectl get pod failing-pod -o yaml > /tmp/pod.yaml
vim /tmp/pod.yaml                        # fix the issue
kubectl delete pod failing-pod
kubectl apply -f /tmp/pod.yaml
```

---

## 12.7 Speed Drills

Practice these until you can do each in under 2 minutes:

### Drill Set A — Pod Operations

```bash
# Timer: < 1 minute each
kubectl run drill1 --image=nginx:alpine -n default
kubectl run drill2 --image=busybox:1.35 --command -- sleep 3600
kubectl run drill3 --image=nginx:alpine --labels="env=test,app=drill"
kubectl get pods drill1 drill2 drill3 -o wide
kubectl exec -it drill2 -- ls /tmp
kubectl delete pod drill1 drill2 drill3
```

### Drill Set B — Deployments

```bash
# Timer: < 2 minutes
kubectl create deployment speed-test --image=nginx:1.23 --replicas=3
kubectl set image deployment/speed-test speed-test=nginx:1.25
kubectl rollout status deployment/speed-test
kubectl rollout undo deployment/speed-test
kubectl scale deployment speed-test --replicas=1
kubectl delete deployment speed-test
```

### Drill Set C — RBAC

```bash
# Timer: < 3 minutes
kubectl create ns rbac-drill
kubectl create sa drillbot -n rbac-drill
kubectl create role podmanager --verb=get,list,create,delete --resource=pods -n rbac-drill
kubectl create rolebinding drillbot-pm --role=podmanager --serviceaccount=rbac-drill:drillbot -n rbac-drill
kubectl auth can-i get pods --as=system:serviceaccount:rbac-drill:drillbot -n rbac-drill
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-drill:drillbot -n rbac-drill
kubectl auth can-i get nodes --as=system:serviceaccount:rbac-drill:drillbot
kubectl delete ns rbac-drill
```

### Drill Set D — Scheduling

```bash
# Timer: < 3 minutes
kubectl label node minikube tier=frontend
kubectl taint node minikube-m02 team=ops:NoSchedule 2>/dev/null || true
# Create pod with affinity for tier=frontend
kubectl run sched-test --image=nginx:alpine \
  --dry-run=client -o yaml | \
  python3 -c "
import sys, yaml
pod = yaml.safe_load(sys.stdin)
pod['spec']['nodeSelector'] = {'tier': 'frontend'}
print(yaml.dump(pod))" | kubectl apply -f -
kubectl get pod sched-test -o wide    # should be on minikube
kubectl delete pod sched-test
kubectl label node minikube tier-
```

---

## 12.8 Full Mock Exam

> Do this untimed first, then timed (aim for 90 minutes or less).

### Task 1 (4%) — Create a multi-container pod

Create a pod named `ambassador` in namespace `default`.
- Container 1: `main`, image `nginx:alpine`, port 80
- Container 2: `sidecar`, image `busybox:1.35`, command: `sleep 3600`
- Both containers share a volume at `/shared`

### Task 2 (6%) — Create a deployment with rolling update

Create a deployment `rolling-test` with image `nginx:1.23`, replicas 4.
Update image to `nginx:1.25`. Verify rollout succeeded.
Roll back to previous version.

### Task 3 (5%) — Create a service and verify DNS

Create a deployment `dns-test` with `nginxdemos/hello:plain-text`.
Expose it as ClusterIP on port 80.
Run a pod and verify you can reach it by DNS name.

### Task 4 (6%) — Configure RBAC

Create namespace `security-test`.
Create ServiceAccount `app-sa` in that namespace.
Create a Role that allows get, list, watch on pods and deployments.
Bind the Role to `app-sa`.
Verify permissions with `kubectl auth can-i`.

### Task 5 (5%) — Create persistent storage

Create a PV `exam-pv` (hostPath `/mnt/exam`, 1Gi, RWO, storageClass `exam-storage`).
Create a PVC `exam-pvc` requesting 500Mi from storageClass `exam-storage`.
Create a pod using this PVC mounted at `/data`.

### Task 6 (4%) — Schedule pod on specific node

Label a node with `workload=critical`.
Create a deployment `critical-app` that only runs on that node using `nodeSelector`.

### Task 7 (6%) — Fix a broken deployment

A deployment named `broken-deploy` exists in namespace `broken-ns`.
It has 0 ready pods. Find and fix the issue.
(Create it broken: `kubectl create deployment broken-deploy --image=nginx:BROKEN -n broken-ns`)

### Task 8 (5%) — Ingress

Install ingress controller.
Create two deployments and services.
Create an Ingress routing `/app1` to service1 and `/app2` to service2.

### Task 9 (6%) — etcd backup

Take a snapshot of etcd and save it to `/tmp/exam-backup.db`.

### Task 10 (4%) — Helm

Install the bitnami/nginx chart with 3 replicas as release `exam-nginx`.
Update to 2 replicas.
Roll back to the original revision.

---

## 12.9 Practice Resources

```
killer.sh            → Official exam simulator (free with exam purchase)
                       Do it at least twice

killercoda.com       → Free Kubernetes labs in browser

kodekloud.com        → CKA practice labs

labs.play-with-k8s.com → Free k8s playground

minikube + local     → Daily practice (this course)
```

---

## 12.10 Day-Before Checklist

```
□ Sleep well — fatigue kills accuracy
□ Bookmark: kubernetes.io/docs/reference/kubectl/cheatsheet/
□ Know your vim/nano basics (edit YAML quickly)
□ Review: etcd backup/restore commands (Module 03)
□ Review: RBAC creation sequence (Module 08)
□ Review: troubleshooting mental model (Module 10)
□ Test your internet and browser setup
□ Have water nearby
```

---

## kubectl Cheatsheet (Quick Reference)

```bash
# Context
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>

# Resources (all common ones)
kubectl get pods,svc,deploy,rs,ds,sts,jobs,cronjobs,ing,pv,pvc,sc,cm,secrets,sa,roles,rolebindings,clusterroles,clusterrolebindings,ns,nodes -o wide

# Imperative create (key ones)
kubectl run NAME --image=IMAGE [flags]
kubectl create deploy NAME --image=IMAGE --replicas=N
kubectl expose deploy NAME --port=P [--type=T]
kubectl create cm NAME --from-literal=k=v
kubectl create secret generic NAME --from-literal=k=v
kubectl create sa NAME
kubectl create role NAME --verb=V --resource=R
kubectl create rolebinding NAME --role=R --user=U
kubectl create ns NAME

# Apply / delete
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl delete pod NAME --grace-period=0 --force   # force delete

# Rollout
kubectl rollout status deploy/NAME
kubectl rollout history deploy/NAME
kubectl rollout undo deploy/NAME
kubectl rollout undo deploy/NAME --to-revision=N

# Scale
kubectl scale deploy NAME --replicas=N

# Drain / cordon
kubectl cordon NODE
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl uncordon NODE

# Debug
kubectl describe RESOURCE NAME
kubectl logs POD [-c CONTAINER] [-f] [-p] [--tail=N]
kubectl exec -it POD -- /bin/sh
kubectl port-forward pod/NAME LOCALPORT:CONTAINERPORT
kubectl top nodes
kubectl top pods
kubectl auth can-i VERB RESOURCE --as=USER
```

---

## Congratulations — You're Ready

Complete all 12 modules, do the mock exam, and practice on killer.sh.

Focus time: **Module 10 (Troubleshooting) > Module 03 (etcd/upgrade) > Module 06 (Networking) > Module 08 (RBAC)**

Good luck with your CKA!
