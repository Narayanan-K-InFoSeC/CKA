# Module 08 — Security

> RBAC and service accounts are guaranteed exam topics. Know them deeply.

---

## 8.1 RBAC — Role-Based Access Control

Controls WHO can do WHAT to WHICH resources.

```
Subject (who)     → User, Group, ServiceAccount
Role (what)       → list of rules (verbs + resources)
RoleBinding       → binds Subject to Role
```

### RBAC Resource Types

| Resource | Scope |
|----------|-------|
| Role | Namespace-scoped |
| ClusterRole | Cluster-wide |
| RoleBinding | Namespace-scoped (binds Role or ClusterRole) |
| ClusterRoleBinding | Cluster-wide |

---

### Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]              # "" = core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

- apiGroups: [""]
  resources: ["pods/log"]      # subresources with /
  verbs: ["get"]

- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["specific-pod"]   # limit to specific resource names
  verbs: ["get", "delete"]
```

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader
  namespace: development
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-service-account
  namespace: development
roleRef:
  kind: Role             # or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-node-reader
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### RBAC Imperative Commands

```bash
# Create role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n development

# Create clusterrole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create rolebinding
kubectl create rolebinding dev-pod-reader \
  --role=pod-reader \
  --user=alice \
  -n development

# Bind clusterrole in a namespace (scoped!)
kubectl create rolebinding alice-admin \
  --clusterrole=admin \
  --user=alice \
  -n development

# Create clusterrolebinding
kubectl create clusterrolebinding bob-node-reader \
  --clusterrole=node-reader \
  --user=bob

# Check permissions (can-i)
kubectl auth can-i list pods --as=alice -n development
kubectl auth can-i delete nodes --as=bob
kubectl auth can-i create deployments --as=system:serviceaccount:default:my-sa

# List what a user can do
kubectl auth can-i --list --as=alice -n development
```

---

## 8.2 Service Accounts

Every pod runs as a service account. Default SA is `default`.

```bash
# List service accounts
kubectl get serviceaccounts
kubectl get sa    # shorthand

# Create service account
kubectl create serviceaccount my-sa
kubectl create sa my-sa -n production

# Describe to see token secret
kubectl describe sa my-sa

# Use in pod
# (auto-mounted at /var/run/secrets/kubernetes.io/serviceaccount/)
```

### Pod using a ServiceAccount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
spec:
  serviceAccountName: my-sa    # use this service account
  automountServiceAccountToken: true   # default is true
  containers:
  - name: app
    image: bitnami/kubectl:latest
    command: ["sh", "-c", "kubectl get pods; sleep 3600"]
```

### Disabling token auto-mount

```yaml
# On the service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-token-sa
automountServiceAccountToken: false

# Or per-pod
spec:
  automountServiceAccountToken: false
```

---

## 8.3 Security Contexts

Run containers with specific user/group/capabilities.

### Pod-level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000             # run all containers as UID 1000
    runAsGroup: 3000            # GID 3000
    runAsNonRoot: true          # fail if image runs as root
    fsGroup: 2000               # volume files owned by GID 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]    # add only what's needed
```

---

## 8.4 Pod Security Standards (PSS)

Kubernetes-native pod security policies (replaced PodSecurityPolicy in 1.25).

### Three Profiles

| Level | Description |
|-------|------------|
| **privileged** | No restrictions |
| **baseline** | Prevents privilege escalation |
| **restricted** | Hardened — follows security best practices |

### Apply via Namespace Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
# Apply PSS to existing namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# Test what PSS violations would happen
kubectl apply -f pod.yaml --dry-run=server -n secure-ns
```

---

## 8.5 TLS Certificates

```
CA (Certificate Authority)  → signs all certificates
├── apiserver.crt           → API server identity
├── apiserver-kubelet-client.crt → apiserver talks to kubelet
├── etcd/server.crt         → etcd server
└── sa.pub                  → service account token signing
```

```bash
# View all certs and expiry
kubeadm certs check-expiration

# Decode a certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text

# Generate a user certificate (manual)
# 1. Generate private key
openssl genrsa -out alice.key 2048

# 2. Generate CSR (Certificate Signing Request)
openssl req -new -key alice.key -out alice.csr \
  -subj "/CN=alice/O=dev-team"

# 3. Sign with cluster CA
openssl x509 -req -in alice.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out alice.crt -days 365
```

### CertificateSigningRequest (Kubernetes API)

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice
spec:
  request: <base64-encoded-csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
```

```bash
# Submit CSR
cat alice.csr | base64 | tr -d '\n'   # encode CSR
kubectl apply -f csr.yaml

# Approve
kubectl certificate approve alice

# Get the signed cert
kubectl get csr alice -o jsonpath='{.status.certificate}' | base64 -d > alice.crt
```

---

## 8.6 kubeconfig & Context Management

```bash
# View current config
kubectl config view
kubectl config view --minify    # current context only

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context production-cluster

# Set default namespace for context
kubectl config set-context --current --namespace=dev

# Add new cluster/user/context
kubectl config set-cluster my-cluster \
  --server=https://192.168.1.100:6443 \
  --certificate-authority=/path/to/ca.crt

kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key

kubectl config set-context alice-context \
  --cluster=my-cluster \
  --user=alice \
  --namespace=development
```

---

## Lab 08-A — RBAC Hands-On

```bash
# 1. Create a namespace and service account
kubectl create namespace team-a
kubectl create serviceaccount developer-sa -n team-a

# 2. Create a Role (read-only pods and deployments)
cat > /tmp/dev-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly
  namespace: team-a
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f /tmp/dev-role.yaml

# 3. Bind the role to the service account
kubectl create rolebinding dev-readonly-binding \
  --role=dev-readonly \
  --serviceaccount=team-a:developer-sa \
  -n team-a

# 4. Create a pod that uses this SA
cat > /tmp/sa-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
  namespace: team-a
spec:
  serviceAccountName: developer-sa
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl apply -f /tmp/sa-pod.yaml

# 5. Deploy something to test against
kubectl create deployment test-app --image=nginx:alpine -n team-a

# 6. Test permissions from inside the pod
kubectl exec -n team-a dev-pod -- kubectl get pods -n team-a      # allowed
kubectl exec -n team-a dev-pod -- kubectl get deployments -n team-a  # allowed
kubectl exec -n team-a dev-pod -- kubectl delete pod test-app-xxxxx -n team-a
# Should get: Error from server (Forbidden)

# 7. Check permissions with auth can-i
kubectl auth can-i get pods --as=system:serviceaccount:team-a:developer-sa -n team-a
kubectl auth can-i delete pods --as=system:serviceaccount:team-a:developer-sa -n team-a
kubectl auth can-i get pods --as=system:serviceaccount:team-a:developer-sa -n default
# First: yes, Second: no, Third: no

kubectl delete namespace team-a
```

---

## Lab 08-B — Security Contexts

```bash
# 1. Run a pod as a specific user
cat > /tmp/user-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: user-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "id; sleep 3600"]
EOF

kubectl apply -f /tmp/user-pod.yaml
kubectl logs user-pod
# Output: uid=1000 gid=3000 groups=2000,3000

# 2. Verify volume files are owned by fsGroup
cat > /tmp/fsgroup-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: fsgroup-pod
spec:
  securityContext:
    fsGroup: 2000
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "ls -la /data && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
EOF

kubectl apply -f /tmp/fsgroup-pod.yaml
kubectl logs fsgroup-pod
# /data owned by group 2000

# 3. Privileged vs restricted
cat > /tmp/priv-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: priv-test
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "cat /proc/1/status | grep CapEff; sleep 3600"]
    securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_ADMIN"]
EOF

kubectl apply -f /tmp/priv-pod.yaml
kubectl logs priv-test    # shows effective capabilities

kubectl delete pod user-pod fsgroup-pod priv-test
```

---

## Lab 08-C — Pod Security Standards

```bash
# 1. Create namespace with baseline policy
kubectl create namespace pss-test
kubectl label namespace pss-test \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# 2. Try to run a privileged pod (should fail)
cat > /tmp/privileged-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-attempt
  namespace: pss-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      privileged: true    # violates baseline
EOF

kubectl apply -f /tmp/privileged-pod.yaml
# Error: violates PodSecurity "baseline"

# 3. Run a compliant pod
cat > /tmp/compliant-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: pss-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
EOF

kubectl apply -f /tmp/compliant-pod.yaml
kubectl get pod compliant-pod -n pss-test   # should be Running

kubectl delete namespace pss-test
```

---

## Checklist

- [ ] Understand Role vs ClusterRole vs RoleBinding vs ClusterRoleBinding
- [ ] Can create roles imperatively with `kubectl create role`
- [ ] Know the `kubectl auth can-i` command
- [ ] Can bind ClusterRole in a namespace using RoleBinding
- [ ] Know what a ServiceAccount is and how pods use them
- [ ] Can set `runAsUser`, `runAsGroup`, `fsGroup` in securityContext
- [ ] Know the three Pod Security Standards (privileged/baseline/restricted)
- [ ] Can manage kubeconfig contexts

---

**Next:** [Module 09 — Logging & Monitoring](./Module-09-Logging-Monitoring.md)
