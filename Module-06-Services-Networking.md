# Module 06 — Services & Networking

> Heavily tested in 2026 exams. Know every Service type, Ingress, NetworkPolicy, and DNS cold.

---

## 6.1 Services

A Service gives a stable IP and DNS name to a set of pods.
It uses **label selectors** to find pods.

### Service Types

| Type | Accessibility | Use Case |
|------|--------------|---------|
| ClusterIP | Inside cluster only | Internal microservices |
| NodePort | Via node IP + port | Dev/testing external access |
| LoadBalancer | Via cloud LB IP | Production external access |
| ExternalName | CNAME DNS alias | External services |

---

### ClusterIP (default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP         # default — can omit this line
  selector:
    app: web              # selects pods with app=web label
  ports:
  - name: http
    port: 80              # service port (what clients use)
    targetPort: 8080      # container port (where pod listens)
    protocol: TCP
```

```bash
# Imperative
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose pod nginx --port=80 --name=nginx-svc

# Get service
kubectl get svc web-service
kubectl describe svc web-service

# Check endpoints (actual pod IPs)
kubectl get endpoints web-service
```

---

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80              # ClusterIP port
    targetPort: 8080      # container port
    nodePort: 30080       # external port (30000-32767)
```

```bash
# Access via node IP
minikube ip                 # get node IP
curl http://$(minikube ip):30080

# Let Kubernetes assign a NodePort (don't specify nodePort)
kubectl expose deployment web --type=NodePort --port=80
```

---

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# On minikube — tunnel needed to get external IP
minikube tunnel    # run in separate terminal

kubectl get svc web-lb    # EXTERNAL-IP will appear
```

---

### ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.prod.example.com
  # No selector needed — just DNS alias
```

---

## 6.2 Headless Services

No ClusterIP — DNS returns pod IPs directly. Used with StatefulSets.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None           # headless!
  selector:
    app: mysql
  ports:
  - port: 3306
```

```bash
# With headless service, DNS query returns all pod IPs
# pod-0.mysql-headless.default.svc.cluster.local → pod-0 IP
# pod-1.mysql-headless.default.svc.cluster.local → pod-1 IP
```

---

## 6.3 CoreDNS & Service Discovery

Every service gets a DNS name automatically:

```
<service-name>.<namespace>.svc.cluster.local

# Examples:
web-service.default.svc.cluster.local
my-db.production.svc.cluster.local
```

Pods in the same namespace can use short names:
```
curl http://web-service        # works within default namespace
curl http://web-service.prod   # works from other namespace (short)
```

```bash
# CoreDNS runs as a Deployment in kube-system
kubectl get deployment -n kube-system coredns
kubectl get configmap -n kube-system coredns -o yaml

# Test DNS from inside a pod
kubectl run dns-test --image=busybox:1.35 --rm -it -- sh
  nslookup kubernetes.default
  nslookup web-service
  nslookup web-service.default.svc.cluster.local
  cat /etc/resolv.conf
  exit
```

---

## 6.4 Ingress

Provides HTTP/HTTPS routing to services — single entry point for multiple services.

```
Internet
    │
    ▼
 Ingress Controller (nginx, traefik, etc.)
    │
    ├── /api  → api-service:8080
    ├── /web  → web-service:80
    └── app2.example.com → app2-service:3000
```

### Install Ingress Controller on Minikube

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

### Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

  # TLS
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls-secret
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress app-ingress
```

### Path Types

| PathType | Behavior |
|----------|---------|
| Exact | Must match exactly: /web ≠ /web/ |
| Prefix | Matches prefix: /web matches /web, /web/ , /web/page |
| ImplementationSpecific | Controller-defined |

---

## 6.5 Network Policies

Control traffic between pods. Without NetworkPolicy, all pods can talk to all pods.

```
Default: ALL pods can communicate with ALL pods
NetworkPolicy: ONLY allowed traffic flows
```

### Deny All (baseline — then open up)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}       # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

### Allow specific ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend      # this policy applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend   # only frontend pods can talk to backend
    - namespaceSelector:
        matchLabels:
          name: monitoring  # monitoring namespace can also access
    ports:
    - protocol: TCP
      port: 8080
```

### Allow specific egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53             # DNS
    - protocol: TCP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

> NetworkPolicies require a CNI that supports them (Calico, Cilium, Weave). Flannel does NOT support NetworkPolicy.

---

## 6.6 CNI Plugins

CNI = Container Network Interface. Provides pod networking.

| CNI | NetworkPolicy | Notes |
|-----|--------------|-------|
| Flannel | No | Simple, no policy support |
| Calico | Yes | Most popular, full policy support |
| Cilium | Yes | eBPF-based, high performance |
| Weave | Yes | Easy setup |

```bash
# Check which CNI is installed
ls /etc/cni/net.d/
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel"
```

---

## 6.7 Gateway API (2026 Exam Relevant)

Next generation replacement for Ingress. More expressive.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

---

## Lab 06-A — ClusterIP & NodePort Services

```bash
# 1. Start minikube
minikube start

# 2. Create a deployment
kubectl create deployment web --image=nginxdemos/hello:plain-text --replicas=3
kubectl get pods -l app=web

# 3. Expose as ClusterIP
kubectl expose deployment web --port=80 --target-port=80 --name=web-clusterip
kubectl get svc web-clusterip
kubectl get endpoints web-clusterip   # shows actual pod IPs

# 4. Test ClusterIP from inside cluster
kubectl run test-client --image=busybox:1.35 --rm -it -- sh
  wget -qO- http://web-clusterip
  nslookup web-clusterip
  exit

# 5. Expose as NodePort
kubectl expose deployment web --port=80 --target-port=80 \
  --type=NodePort --name=web-nodeport

kubectl get svc web-nodeport    # see the NodePort assigned
NODEPORT=$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')

# Access via minikube
curl http://$(minikube ip):$NODEPORT

# 6. Scale and verify load balancing
kubectl scale deployment web --replicas=5
kubectl run load-test --image=busybox:1.35 --rm -it -- sh
  for i in $(seq 1 10); do wget -qO- http://web-clusterip | grep "Server name"; done
  exit
# Should see different pod names (load balancing working)

kubectl delete svc web-clusterip web-nodeport
kubectl delete deployment web
```

---

## Lab 06-B — CoreDNS & Service Discovery

```bash
# 1. Create services in different namespaces
kubectl create namespace backend
kubectl create namespace frontend

kubectl create deployment api --image=nginx:alpine -n backend
kubectl expose deployment api --port=80 -n backend

kubectl create deployment ui --image=nginx:alpine -n frontend
kubectl expose deployment ui --port=80 -n frontend

# 2. Test DNS resolution from a pod
kubectl run dns-debugger --image=busybox:1.35 --rm -it -- sh

  # Short form (same namespace)
  nslookup ui                              # won't work — different namespace

  # Full DNS names
  nslookup api.backend.svc.cluster.local
  nslookup ui.frontend.svc.cluster.local

  # Partial (cross-namespace short form)
  nslookup api.backend

  # kubernetes service
  nslookup kubernetes.default.svc.cluster.local

  cat /etc/resolv.conf
  # search default.svc.cluster.local svc.cluster.local cluster.local
  exit

# 3. Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# 4. Check CoreDNS logs
kubectl logs -n kube-system deployment/coredns

kubectl delete namespace backend frontend
```

---

## Lab 06-C — Ingress with Minikube

```bash
# 1. Enable ingress addon
minikube addons enable ingress

# Wait for ingress controller
kubectl get pods -n ingress-nginx -w
# Wait until nginx-ingress-controller is Running

# 2. Create two apps
kubectl create deployment web1 --image=nginxdemos/hello:plain-text
kubectl create deployment web2 --image=hashicorp/http-echo -- -text="App2 Response"

kubectl expose deployment web1 --port=80
kubectl expose deployment web2 --port=5678

# 3. Create Ingress
cat > /tmp/app-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /web1
        pathType: Prefix
        backend:
          service:
            name: web1
            port:
              number: 80
      - path: /web2
        pathType: Prefix
        backend:
          service:
            name: web2
            port:
              number: 5678
EOF

kubectl apply -f /tmp/app-ingress.yaml

# 4. Get minikube IP and add to /etc/hosts
MINIKUBE_IP=$(minikube ip)
echo "${MINIKUBE_IP} app.local" | sudo tee -a /etc/hosts

# 5. Test
curl http://app.local/web1
curl http://app.local/web2

# 6. View ingress details
kubectl describe ingress app-ingress

# Cleanup
kubectl delete ingress app-ingress
kubectl delete deployment web1 web2
kubectl delete svc web1 web2
# Remove from /etc/hosts
sudo sed -i '' '/app.local/d' /etc/hosts
```

---

## Lab 06-D — Network Policies

```bash
# 1. For NetworkPolicy support, install Calico
minikube start --network-plugin=cni --cni=calico

# Wait for Calico pods
kubectl get pods -n kube-system | grep calico

# 2. Create test namespace and pods
kubectl create namespace netpol-test

kubectl run frontend --image=nginx:alpine -n netpol-test \
  --labels="app=frontend"
kubectl run backend  --image=nginx:alpine -n netpol-test \
  --labels="app=backend"
kubectl run database --image=nginx:alpine -n netpol-test \
  --labels="app=database"

kubectl expose pod backend  --port=80 -n netpol-test
kubectl expose pod database --port=80 -n netpol-test

# 3. Without NetworkPolicy — everything can talk
kubectl exec -n netpol-test frontend -- wget -qO- http://backend
kubectl exec -n netpol-test frontend -- wget -qO- http://database

# 4. Apply deny-all policy
cat > /tmp/deny-all.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

kubectl apply -f /tmp/deny-all.yaml

# 5. Now nothing can talk
kubectl exec -n netpol-test frontend -- wget -qO- --timeout=3 http://backend
# Should timeout

# 6. Allow only frontend → backend
cat > /tmp/allow-frontend-backend.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
EOF

kubectl apply -f /tmp/allow-frontend-backend.yaml

# Also need egress from frontend to backend, and DNS
cat > /tmp/allow-frontend-egress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - port: 80
  - ports:              # allow DNS
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF

kubectl apply -f /tmp/allow-frontend-egress.yaml

# 7. Test: frontend can reach backend
kubectl exec -n netpol-test frontend -- wget -qO- --timeout=5 http://backend
# Should succeed

# 8. Test: frontend cannot reach database
kubectl exec -n netpol-test frontend -- wget -qO- --timeout=3 http://database
# Should timeout

kubectl delete namespace netpol-test
```

---

## DNS Troubleshooting Quick Reference

```bash
# Check if CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from debug pod
kubectl run dns-debug --image=busybox:1.35 --rm -it -- nslookup kubernetes

# Check CoreDNS configmap
kubectl get cm coredns -n kube-system -o yaml

# Check service FQDN format
# <service>.<namespace>.svc.<cluster-domain>
# Default cluster domain: cluster.local
```

---

## Checklist

- [ ] Know all 4 Service types and when to use each
- [ ] Can create Services imperatively with `kubectl expose`
- [ ] Understand ClusterIP, NodePort, LoadBalancer differences
- [ ] Know the DNS name format: `<svc>.<ns>.svc.cluster.local`
- [ ] Can create Ingress rules with path and host routing
- [ ] Can write NetworkPolicy to deny-all and allow specific traffic
- [ ] Know that Flannel does NOT support NetworkPolicy

---

**Next:** [Module 07 — Storage](./Module-07-Storage.md)
