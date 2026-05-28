# Module 11 — Helm & Package Management

> Helm tasks appear more often in recent exams. Know install, upgrade, rollback, and customization.

---

## 11.1 What is Helm?

Helm is the **package manager for Kubernetes**.

```
Chart     = package (like a .deb or .rpm)
Release   = installed instance of a chart
Repository = collection of charts (like apt/brew)
values.yaml = configuration file for a chart
```

### Helm vs kubectl

| kubectl | Helm |
|---------|------|
| Apply individual YAML files | Install entire applications |
| Manual versioning | Built-in versioning & rollback |
| No templating | Go template engine |
| Manual config | values.yaml configuration |

---

## 11.2 Helm Architecture

```
helm CLI
    │
    ├── Helm Repositories (ArtifactHub, Bitnami, etc.)
    │       └── Charts (nginx, wordpress, prometheus...)
    │
    └── Kubernetes Cluster
            └── Releases (installed chart instances)
```

Helm 3 (current) stores release information in **Kubernetes Secrets** (namespace-scoped).

---

## 11.3 Essential Helm Commands

### Repository Management

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# List repositories
helm repo list

# Update repository cache
helm repo update

# Search for charts
helm search repo nginx
helm search repo bitnami/nginx
helm search hub nginx             # search ArtifactHub (global)

# Get chart details
helm show chart bitnami/nginx
helm show values bitnami/nginx    # default values
helm show readme bitnami/nginx
```

### Installing Charts

```bash
# Basic install
helm install my-nginx bitnami/nginx

# Install in a specific namespace
helm install my-nginx bitnami/nginx -n webapps --create-namespace

# Install with custom values
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Install with values file
helm install my-nginx bitnami/nginx -f my-values.yaml

# Install specific version
helm install my-nginx bitnami/nginx --version 15.0.0

# Dry run (preview manifests)
helm install my-nginx bitnami/nginx --dry-run
helm install my-nginx bitnami/nginx --debug --dry-run | head -50

# Generate YAML without installing (template)
helm template my-nginx bitnami/nginx
helm template my-nginx bitnami/nginx -f values.yaml
```

### Managing Releases

```bash
# List releases
helm list
helm list -n webapps               # in a namespace
helm list -A                        # all namespaces

# Get release status
helm status my-nginx

# Get release values
helm get values my-nginx
helm get values my-nginx --all     # all values including defaults

# Get rendered manifests
helm get manifest my-nginx

# Get all release info
helm get all my-nginx
```

### Upgrading

```bash
# Upgrade a release (new values)
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=5

# Upgrade with new values file
helm upgrade my-nginx bitnami/nginx -f new-values.yaml

# Upgrade to newer chart version
helm upgrade my-nginx bitnami/nginx --version 16.0.0

# Install if not exists, upgrade if exists
helm upgrade --install my-nginx bitnami/nginx -f values.yaml

# Upgrade history
helm history my-nginx
```

### Rollback

```bash
# Rollback to previous release
helm rollback my-nginx

# Rollback to specific revision
helm rollback my-nginx 2

# View history
helm history my-nginx
```

### Uninstalling

```bash
# Uninstall release
helm uninstall my-nginx
helm uninstall my-nginx -n webapps

# Keep history (can rollback after)
helm uninstall my-nginx --keep-history
```

---

## 11.4 Chart Structure

```
my-chart/
├── Chart.yaml            # chart metadata
├── values.yaml           # default configuration
├── charts/               # chart dependencies
└── templates/            # Kubernetes YAML templates
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── _helpers.tpl       # template helper functions
    └── NOTES.txt          # post-install notes
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application chart
type: application
version: 1.0.0           # chart version
appVersion: "2.1.0"      # app version being packaged
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  hosts:
  - host: chart-example.local
    paths:
    - path: /
      pathType: ImplementationSpecific

resources:
  limits:
    cpu: 500m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

---

## 11.5 Creating Your Own Chart

```bash
# Scaffold a new chart
helm create my-app

# Validate chart syntax
helm lint my-app/

# Package into .tgz
helm package my-app/

# Install from local directory
helm install test-release ./my-app/

# Install from local package
helm install test-release my-app-1.0.0.tgz
```

---

## Lab 11-A — Install and Manage a Release

```bash
# 1. Add bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Search for available charts
helm search repo bitnami | head -20
helm search repo bitnami/nginx

# 3. Look at default values
helm show values bitnami/nginx | head -50

# 4. Install nginx with custom values
helm install my-nginx bitnami/nginx \
  --set replicaCount=2 \
  --set service.type=NodePort \
  --namespace default

# 5. Check the release
helm list
helm status my-nginx
kubectl get all -l app.kubernetes.io/instance=my-nginx

# 6. Get the values used
helm get values my-nginx

# 7. Access the service
NODEPORT=$(kubectl get svc my-nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$(minikube ip):$NODEPORT

# 8. Upgrade with new replicas
helm upgrade my-nginx bitnami/nginx --set replicaCount=3 --reuse-values
kubectl get pods -l app.kubernetes.io/instance=my-nginx

# 9. Check history
helm history my-nginx

# 10. Rollback to revision 1
helm rollback my-nginx 1
kubectl get pods -l app.kubernetes.io/instance=my-nginx
# Should be back to 2 replicas

# Cleanup
helm uninstall my-nginx
```

---

## Lab 11-B — Custom Values File

```bash
# 1. Create a custom values file
cat > /tmp/my-nginx-values.yaml << 'EOF'
replicaCount: 3

service:
  type: NodePort
  nodePorts:
    http: "30100"

resources:
  limits:
    cpu: 300m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi
EOF

# 2. Preview what will be deployed
helm template my-nginx bitnami/nginx -f /tmp/my-nginx-values.yaml | head -80

# 3. Install with values file
helm install my-nginx bitnami/nginx -f /tmp/my-nginx-values.yaml

# 4. Verify values were applied
helm get values my-nginx

kubectl get deployment my-nginx -o jsonpath='{.spec.replicas}'
# Should be 3

# 5. Update values file and upgrade
sed -i '' 's/replicaCount: 3/replicaCount: 4/' /tmp/my-nginx-values.yaml
helm upgrade my-nginx bitnami/nginx -f /tmp/my-nginx-values.yaml

kubectl get pods -l app.kubernetes.io/instance=my-nginx
# Should be 4 pods

helm uninstall my-nginx
```

---

## Lab 11-C — Create a Simple Chart

```bash
# 1. Create chart skeleton
helm create webapp
ls webapp/

# 2. Look at the structure
tree webapp/ 2>/dev/null || find webapp -type f

# 3. Customize values.yaml
cat > webapp/values.yaml << 'EOF'
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "alpine"

service:
  type: NodePort
  port: 80

ingress:
  enabled: false

resources:
  limits:
    cpu: 200m
    memory: 64Mi
  requests:
    cpu: 50m
    memory: 32Mi

autoscaling:
  enabled: false
EOF

# 4. Lint the chart
helm lint webapp/

# 5. Template to preview output
helm template my-webapp webapp/ | head -60

# 6. Install
helm install my-webapp webapp/

# 7. Verify
kubectl get all -l app.kubernetes.io/instance=my-webapp
helm status my-webapp

# 8. Upgrade with --set
helm upgrade my-webapp webapp/ --set replicaCount=3
kubectl get pods -l app.kubernetes.io/instance=my-webapp

# 9. Package
helm package webapp/
ls *.tgz

helm uninstall my-webapp
rm -f webapp-1.0.0.tgz
rm -rf webapp/
```

---

## Checklist

- [ ] Can add Helm repos and search for charts
- [ ] Can install charts with `--set` and `-f values.yaml`
- [ ] Can list, status, and history releases
- [ ] Can upgrade a release with new values
- [ ] Can rollback a release to a specific revision
- [ ] Can do `helm upgrade --install` (upsert pattern)
- [ ] Know the chart directory structure
- [ ] Can create a basic chart with `helm create`
- [ ] Can lint and template a chart

---

**Next:** [Module 12 — Exam Preparation](./Module-12-Exam-Preparation.md)
