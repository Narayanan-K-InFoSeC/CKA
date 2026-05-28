# Module 04 — Kubernetes Core Objects

> The building blocks. You'll use every one of these in the exam.

---

## 4.1 Pods

A Pod is the smallest deployable unit. One or more containers sharing:
- Same network namespace (localhost)
- Same storage volumes
- Same lifecycle

### Pod YAML Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: web
    env: prod
  annotations:
    description: "Production web server"
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    env:
    - name: ENV_VAR
      value: "production"
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    emptyDir: {}
  restartPolicy: Always    # Always | Never | OnFailure
```

### Imperative Pod Commands (learn these for speed)

```bash
# Create pod quickly
kubectl run nginx --image=nginx:alpine
kubectl run nginx --image=nginx:alpine --port=80

# Generate YAML without creating
kubectl run nginx --image=nginx:alpine --dry-run=client -o yaml

# Create with env var
kubectl run nginx --image=nginx:alpine --env="APP=web" --env="ENV=prod"

# Create with labels
kubectl run nginx --image=nginx:alpine --labels="app=web,tier=frontend"

# Create pod in a namespace
kubectl run nginx --image=nginx:alpine -n my-namespace

# Exec into running pod
kubectl exec -it nginx -- bash
kubectl exec -it nginx -- sh        # if no bash

# Exec single command
kubectl exec nginx -- cat /etc/nginx/nginx.conf

# Copy files
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./myfile.txt nginx:/tmp/myfile.txt
```

### Multi-Container Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: main-app
    image: nginx:alpine
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx

  - name: log-sidecar           # sidecar pattern
    image: busybox:1.35
    command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx

  volumes:
  - name: shared-logs
    emptyDir: {}
```

---

## 4.2 ReplicaSets

Ensures N copies of a pod are always running.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend           # must match template labels
  template:
    metadata:
      labels:
        app: frontend         # must match selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
kubectl create -f replicaset.yaml
kubectl get rs
kubectl describe rs frontend-rs
kubectl scale rs frontend-rs --replicas=5
kubectl delete rs frontend-rs
```

> In practice, always use Deployments (which manage ReplicaSets for you).

---

## 4.3 Deployments

Wraps ReplicaSets, adds rolling updates and rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # max extra pods during update
      maxUnavailable: 0  # no downtime
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.24
        ports:
        - containerPort: 80
```

### Deployment Commands

```bash
# Imperative create
kubectl create deployment web --image=nginx:1.24 --replicas=3

# Generate YAML
kubectl create deployment web --image=nginx:1.24 --replicas=3 \
  --dry-run=client -o yaml > deploy.yaml

# Update image (triggers rolling update)
kubectl set image deployment/web web=nginx:1.25

# Watch rollout
kubectl rollout status deployment/web

# Rollout history
kubectl rollout history deployment/web

# Rollback to previous version
kubectl rollout undo deployment/web

# Rollback to specific revision
kubectl rollout undo deployment/web --to-revision=2

# Scale
kubectl scale deployment web --replicas=5

# Pause/resume rollout
kubectl rollout pause deployment/web
kubectl rollout resume deployment/web
```

---

## 4.4 StatefulSets

Like Deployments but for stateful applications. Provides:
- **Stable, unique pod names** (pod-0, pod-1, pod-2)
- **Ordered deployment and scaling** (0 → 1 → 2)
- **Stable persistent storage** (PVC per pod)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"          # headless service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:         # creates PVC per pod
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

---

## 4.5 DaemonSets

Runs exactly one pod per node (or per selected nodes).

Use cases:
- Log collectors (fluentd, filebeat)
- Node monitoring (node-exporter)
- kube-proxy itself is a DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.16
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

```bash
kubectl get daemonset -n kube-system    # see kube-proxy, etc.
kubectl describe daemonset log-collector
```

---

## 4.6 Jobs

Run a pod to completion. Pod is NOT restarted after success.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  completions: 3          # run 3 times total
  parallelism: 2          # run 2 at a time
  backoffLimit: 4         # retry failed pods 4 times
  template:
    spec:
      restartPolicy: Never    # must be Never or OnFailure
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

```bash
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(2000)"
kubectl get jobs
kubectl get pods      # watch completion
kubectl logs job/pi
```

---

## 4.7 CronJobs

Scheduled Jobs — creates a Job on a cron schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"   # every minute
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.35
            command: ["sh", "-c", "date; echo Hello from CKA"]
  concurrencyPolicy: Forbid    # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

```bash
kubectl create cronjob hello --image=busybox \
  --schedule="*/1 * * * *" -- sh -c "date; echo hello"

kubectl get cronjob
kubectl get jobs    # watch jobs being created
```

---

## 4.8 Labels, Selectors, Annotations

### Labels — for selection and grouping

```bash
# Add label
kubectl label pod my-pod env=production

# Remove label
kubectl label pod my-pod env-

# Overwrite label
kubectl label pod my-pod env=staging --overwrite

# Filter by label
kubectl get pods -l app=web
kubectl get pods -l app=web,env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l 'env notin (dev)'
kubectl get pods -l '!env'              # no env label
```

### Annotations — for metadata only (not selectors)

```bash
# Add annotation
kubectl annotate pod my-pod description="This is a test pod"

# View annotations
kubectl describe pod my-pod | grep -A5 "Annotations:"
```

---

## 4.9 Namespaces

```bash
# Create
kubectl create namespace dev
kubectl create namespace staging

# List
kubectl get namespaces

# Work in a namespace
kubectl get pods -n dev
kubectl get all -n kube-system

# Set default namespace for current context
kubectl config set-context --current --namespace=dev

# Cross-namespace DNS: service-name.namespace.svc.cluster.local
curl http://my-service.dev.svc.cluster.local

# Delete namespace (deletes everything inside!)
kubectl delete namespace dev
```

---

## Lab 04-A — Pod Fundamentals

```bash
# 1. Start minikube
minikube start

# 2. Create a pod imperatively
kubectl run web --image=nginx:1.25 --port=80 --labels="app=web,env=prod"

# 3. Watch it start
kubectl get pod web -w

# 4. Describe the pod
kubectl describe pod web

# 5. Execute into it
kubectl exec -it web -- bash
  nginx -v
  cat /etc/nginx/nginx.conf | head -20
  exit

# 6. Port forward to test
kubectl port-forward pod/web 8080:80 &
curl http://localhost:8080
kill %1

# 7. Get YAML of running pod
kubectl get pod web -o yaml

# 8. Create multi-container pod
cat > /tmp/multi-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: multi
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: shared
      mountPath: /usr/share/nginx/html
  - name: writer
    image: busybox:1.35
    command: ["sh", "-c", "while true; do echo $(date) > /data/index.html; sleep 5; done"]
    volumeMounts:
    - name: shared
      mountPath: /data
  volumes:
  - name: shared
    emptyDir: {}
EOF

kubectl apply -f /tmp/multi-pod.yaml

# 9. Check logs from each container
kubectl logs multi -c app
kubectl logs multi -c writer

# 10. Port forward and verify dynamic content
kubectl port-forward pod/multi 9090:80 &
sleep 6
curl http://localhost:9090    # should show date output
kill %1

# Cleanup
kubectl delete pod web multi
```

---

## Lab 04-B — Deployments & Rolling Updates

```bash
# 1. Create deployment
kubectl create deployment webapp --image=nginx:1.23 --replicas=3

# 2. Watch pods
kubectl get pods -l app=webapp -w &
WATCH_PID=$!

# 3. Update image (triggers rolling update)
kubectl set image deployment/webapp webapp=nginx:1.25

# 4. Watch rollout
kubectl rollout status deployment/webapp

# 5. Stop the watch
kill $WATCH_PID

# 6. Check rollout history
kubectl rollout history deployment/webapp

# 7. Annotate for better history (good exam practice)
kubectl set image deployment/webapp webapp=nginx:1.24
kubectl rollout annotate deployment/webapp kubernetes.io/change-cause="downgraded to 1.24"

kubectl rollout history deployment/webapp

# 8. Rollback to previous
kubectl rollout undo deployment/webapp

kubectl get pods -l app=webapp    # verify 1.25 again

# 9. Scale up
kubectl scale deployment webapp --replicas=5
kubectl get pods -l app=webapp

# 10. Update strategy — Recreate (takes downtime)
kubectl patch deployment webapp -p \
  '{"spec":{"strategy":{"type":"Recreate"}}}'

kubectl set image deployment/webapp webapp=nginx:alpine
kubectl get pods -l app=webapp -w    # see all pods terminate then restart

# Cleanup
kubectl delete deployment webapp
```

---

## Lab 04-C — Jobs & CronJobs

```bash
# 1. Create a Job
kubectl create job countdown --image=busybox \
  -- sh -c "for i in 10 9 8 7 6 5 4 3 2 1; do echo \$i; sleep 1; done; echo Done!"

# 2. Watch Job pods
kubectl get pods -l job-name=countdown -w

# 3. Get logs when complete
kubectl logs job/countdown

# 4. Check Job status
kubectl describe job countdown

# 5. Create a parallel Job
cat > /tmp/parallel-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-work
spec:
  completions: 6
  parallelism: 3
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.35
        command: ["sh", "-c", "echo Processing item $RANDOM; sleep 3; echo Done"]
EOF

kubectl apply -f /tmp/parallel-job.yaml
kubectl get pods -l job-name=parallel-work -w

# 6. Create a CronJob
kubectl create cronjob reporter --image=busybox \
  --schedule="*/2 * * * *" \
  -- sh -c "echo Report generated at $(date)"

# Wait 2 minutes then check
kubectl get cronjob reporter
kubectl get jobs

# Force a manual run
kubectl create job --from=cronjob/reporter reporter-manual-001
kubectl logs job/reporter-manual-001

# Cleanup
kubectl delete job countdown parallel-work reporter-manual-001
kubectl delete cronjob reporter
```

---

## Lab 04-D — Labels and Namespaces

```bash
# 1. Create namespaces
kubectl create namespace dev
kubectl create namespace prod

# 2. Deploy apps in different namespaces
kubectl create deployment web --image=nginx:alpine -n dev --replicas=2
kubectl create deployment web --image=nginx:alpine -n prod --replicas=3

kubectl get deployments -n dev
kubectl get deployments -n prod

# 3. Label pods
kubectl run alpha --image=nginx:alpine --labels="app=alpha,tier=frontend,env=dev"
kubectl run beta  --image=nginx:alpine --labels="app=beta,tier=backend,env=dev"
kubectl run gamma --image=nginx:alpine --labels="app=gamma,tier=frontend,env=prod"

# 4. Select by label
kubectl get pods -l tier=frontend
kubectl get pods -l 'env in (dev,prod)'
kubectl get pods -l tier=frontend,env=dev

# 5. Set context to dev namespace
kubectl config set-context --current --namespace=dev
kubectl get pods     # only dev pods now
kubectl config set-context --current --namespace=default

# Cleanup
kubectl delete pod alpha beta gamma
kubectl delete namespace dev prod
```

---

## Checklist

- [ ] Can create pods imperatively and from YAML
- [ ] Can exec into pods and copy files
- [ ] Can create, update, and rollback Deployments
- [ ] Understand ReplicaSet vs Deployment vs StatefulSet vs DaemonSet
- [ ] Can create Jobs with completions and parallelism
- [ ] Can create CronJobs and manually trigger them
- [ ] Can use label selectors to filter resources
- [ ] Can work across namespaces

---

**Next:** [Module 05 — Scheduling](./Module-05-Scheduling.md)
