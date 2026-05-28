# Module 07 — Storage

> Understand the PV/PVC lifecycle, StorageClasses, and how to debug storage issues.

---

## 7.1 Storage Hierarchy

```
StorageClass   → defines how storage is provisioned (AWS EBS, NFS, local, etc.)
      │
      ▼
PersistentVolume (PV)   → actual storage resource (admin creates or auto-provisioned)
      │
      ▼
PersistentVolumeClaim (PVC)  → request for storage (developer creates)
      │
      ▼
Pod (volumeMounts)           → uses the PVC
```

---

## 7.2 Volume Types (common ones)

| Type | Description |
|------|------------|
| `emptyDir` | Temporary, deleted with pod |
| `hostPath` | Mounts host filesystem path (not for prod) |
| `configMap` | Mounts ConfigMap as files |
| `secret` | Mounts Secret as files |
| `persistentVolumeClaim` | Uses a PVC |
| `nfs` | NFS server mount |
| `awsElasticBlockStore` | AWS EBS (legacy) |
| `csi` | CSI driver — modern standard |

---

## 7.3 PersistentVolume (PV)

Cluster-scoped resource. Pre-provisioned by admin.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local-001
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain    # or Delete, Recycle
  storageClassName: manual
  hostPath:                                 # for local testing only
    path: /mnt/data
```

### Access Modes

| Mode | Short | Multiple pods |
|------|-------|--------------|
| ReadWriteOnce | RWO | One node reads/writes |
| ReadOnlyMany | ROX | Many nodes read-only |
| ReadWriteMany | RWX | Many nodes read/write |
| ReadWriteOncePod | RWOP | One pod reads/writes (k8s 1.22+) |

### Reclaim Policies

| Policy | After PVC deleted |
|--------|-----------------|
| Retain | PV kept, must manually clean up |
| Delete | PV and underlying storage deleted |
| Recycle | (deprecated) Basic scrub then re-available |

### PV Phases

```
Available → Bound → Released → (Retain: stays Released | Delete: deleted)
```

---

## 7.4 PersistentVolumeClaim (PVC)

Namespace-scoped. Developer requests storage.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual     # must match PV's storageClassName
  resources:
    requests:
      storage: 2Gi             # must be ≤ PV capacity
```

### PVC Binding Rules

1. `storageClassName` must match
2. Access mode must be compatible
3. Requested size ≤ PV capacity
4. Label selectors on PVC must match PV labels (optional)

---

## 7.5 Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: storage
      mountPath: /data        # where PVC is mounted inside container
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc       # reference to PVC
```

---

## 7.6 StorageClass & Dynamic Provisioning

With a StorageClass, PVs are created automatically when a PVC is created.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: docker.io/hostpath     # for local dev
# provisioner: kubernetes.io/aws-ebs  # for AWS
# provisioner: pd.csi.storage.gke.io  # for GKE
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer    # or Immediate
```

### Default StorageClass

```bash
# List storage classes
kubectl get storageclass
# or:
kubectl get sc

# Mark a StorageClass as default
kubectl patch storageclass standard \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Check default
kubectl get sc | grep default
```

### PVC with dynamic provisioning

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard    # this is auto-provisioned
  resources:
    requests:
      storage: 1Gi
# PV will be created automatically
```

---

## 7.7 Volume Expansion

```yaml
# StorageClass must have allowVolumeExpansion: true
# Edit the PVC:
kubectl edit pvc my-pvc
# Change resources.requests.storage from 1Gi to 2Gi

# Or with patch:
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# Check status
kubectl describe pvc my-pvc    # look for "Normal  Resizing" events
```

---

## 7.8 ConfigMaps

Non-sensitive configuration data.

```bash
# Create from literal
kubectl create configmap app-config --from-literal=DB_HOST=mysql --from-literal=DB_PORT=3306

# Create from file
kubectl create configmap nginx-config --from-file=nginx.conf

# Create from directory
kubectl create configmap app-configs --from-file=./config-dir/

# View
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

### Use ConfigMap as Env Vars

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-env-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    envFrom:                      # inject ALL keys as env vars
    - configMapRef:
        name: app-config
```

### Use ConfigMap as Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
      # Each key becomes a file: /etc/config/DB_HOST, /etc/config/DB_PORT
```

---

## 7.9 Secrets

Like ConfigMaps but for sensitive data. Base64-encoded (NOT encrypted by default).

```bash
# Create generic secret
kubectl create secret generic db-secret \
  --from-literal=password=MyS3cr3t \
  --from-literal=username=admin

# Create TLS secret
kubectl create secret tls my-tls \
  --cert=server.crt --key=server.key

# Create docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# View (base64 encoded)
kubectl get secret db-secret -o yaml

# Decode
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### Use Secret as Env Var

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

### Use Secret as Volume

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-secret
# Files: /etc/secrets/password, /etc/secrets/username
```

---

## Lab 07-A — PV, PVC, and Pod Storage

```bash
# 1. SSH into minikube to create a host path
minikube ssh
sudo mkdir -p /mnt/pv-data
echo "Hello from PV storage" | sudo tee /mnt/pv-data/test.txt
exit

# 2. Create a PersistentVolume
cat > /tmp/pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  labels:
    type: local
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/pv-data
EOF

kubectl apply -f /tmp/pv.yaml
kubectl get pv
kubectl describe pv local-pv

# 3. Create a PersistentVolumeClaim
cat > /tmp/pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi   # less than PV capacity
EOF

kubectl apply -f /tmp/pvc.yaml
kubectl get pvc local-pvc   # should be Bound

# 4. Create a Pod using the PVC
cat > /tmp/pvc-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "cat /storage/test.txt && echo 'Pod was here' >> /storage/test.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: local-pvc
EOF

kubectl apply -f /tmp/pvc-pod.yaml
kubectl get pod storage-pod

# 5. Check the pod read the file
kubectl logs storage-pod    # should show "Hello from PV storage"

# 6. Write to it and verify persistence
kubectl exec storage-pod -- sh -c "echo 'Updated at: $(date)' >> /storage/test.txt"

# 7. Delete pod and recreate — data persists
kubectl delete pod storage-pod
kubectl apply -f /tmp/pvc-pod.yaml
kubectl exec storage-pod -- cat /storage/test.txt
# Should show all three lines!

# 8. Check PV remains Bound even after pod deletion
kubectl get pv local-pv
kubectl get pvc local-pvc

kubectl delete pod storage-pod pvc local-pvc pv local-pv
```

---

## Lab 07-B — Dynamic Provisioning with StorageClass

```bash
# 1. Check available storage classes
kubectl get storageclass

# On minikube, there's usually a 'standard' default StorageClass
kubectl describe storageclass standard

# 2. Create a PVC without specifying storageClassName (uses default)
cat > /tmp/dynamic-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
EOF

kubectl apply -f /tmp/dynamic-pvc.yaml

# 3. Check PV was auto-created
kubectl get pvc dynamic-pvc    # Bound
kubectl get pv                 # auto-created PV

# 4. Use it in a pod
cat > /tmp/dynamic-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo 'Dynamic storage works!' > /data/proof.txt && cat /data/proof.txt && sleep 3600"]
    volumeMounts:
    - name: dynamic-storage
      mountPath: /data
  volumes:
  - name: dynamic-storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
EOF

kubectl apply -f /tmp/dynamic-pod.yaml
kubectl logs dynamic-pod

kubectl delete pod dynamic-pod pvc dynamic-pvc
```

---

## Lab 07-C — ConfigMaps and Secrets

```bash
# 1. Create a ConfigMap
kubectl create configmap app-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432 \
  --from-literal=APP_ENV=production

kubectl get configmap app-config -o yaml

# 2. Create a config file and load it
cat > /tmp/nginx.conf << 'EOF'
server {
  listen 8080;
  server_name localhost;
  location / {
    root /usr/share/nginx/html;
  }
}
EOF

kubectl create configmap nginx-config --from-file=/tmp/nginx.conf
kubectl describe configmap nginx-config

# 3. Use ConfigMap as env vars
cat > /tmp/cm-env-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-pod
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo DB=$DB_HOST:$DB_PORT ENV=$APP_ENV; sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl apply -f /tmp/cm-env-pod.yaml
kubectl logs cm-env-pod

# 4. Create a Secret
kubectl create secret generic db-credentials \
  --from-literal=username=dbuser \
  --from-literal=password='S3cr3tP@ss!'

# 5. View secret (base64)
kubectl get secret db-credentials -o yaml

# 6. Decode password
kubectl get secret db-credentials \
  -o jsonpath='{.data.password}' | base64 -d
echo    # newline

# 7. Use secret in pod
cat > /tmp/secret-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo User=$DB_USER Pass=$DB_PASS; sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

kubectl apply -f /tmp/secret-pod.yaml
kubectl logs secret-pod

kubectl delete pod cm-env-pod secret-pod
kubectl delete configmap app-config nginx-config
kubectl delete secret db-credentials
```

---

## Troubleshooting Storage Issues

```bash
# PVC stuck in Pending
kubectl describe pvc my-pvc
# Common causes:
# - No matching PV (check storageClassName, accessMode, size)
# - StorageClass provisioner not available
# - Node has no available disk space

# Pod stuck with volume not mounted
kubectl describe pod my-pod
# Look for: "Unable to attach or mount volumes"
# Events: "FailedMount", "FailedAttachVolume"

# Check if node has the PV
kubectl get pv -o wide    # see which node

# Force delete stuck PVC (last resort)
kubectl patch pvc my-pvc -p '{"metadata":{"finalizers":null}}'
```

---

## Checklist

- [ ] Understand PV → PVC → Pod relationship
- [ ] Know all 4 access modes (RWO, ROX, RWX, RWOP)
- [ ] Know all 3 reclaim policies (Retain, Delete, Recycle)
- [ ] Can create PVs and PVCs manually and bind them
- [ ] Know how dynamic provisioning works via StorageClass
- [ ] Can create ConfigMaps and use them as env vars and volumes
- [ ] Can create Secrets and decode them
- [ ] Can troubleshoot Pending PVCs

---

**Next:** [Module 08 — Security](./Module-08-Security.md)
