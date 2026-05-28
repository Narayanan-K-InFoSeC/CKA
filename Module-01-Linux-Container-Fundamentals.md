# Module 01 — Linux & Container Fundamentals

> Foundation before Kubernetes. If you are comfortable with Linux and Docker, skim the theory and go straight to the labs.

---

## 1.1 Linux CLI Basics

### Navigation & File Management

```bash
pwd                        # current directory
ls -lah                    # list with human-readable sizes
cd /etc                    # change directory
mkdir -p /tmp/test/sub     # create nested dirs
cp source dest             # copy
mv old new                 # move / rename
rm -rf dir/                # delete directory
find /etc -name "*.conf"   # find files by name
find / -type f -size +10M  # find large files
```

### File Permissions

```bash
# Format: [type][owner][group][others]
# -rwxr-xr-- = file, owner=rwx, group=r-x, others=r--

chmod 755 script.sh        # rwxr-xr-x
chmod u+x script.sh        # add execute for owner
chown user:group file      # change owner
ls -la                     # view permissions
```

### Text Processing (critical for CKA)

```bash
grep "error" /var/log/syslog            # search in file
grep -r "pattern" /etc/                 # recursive search
grep -v "comment" file                  # invert match
awk '{print $1, $3}' file               # print columns 1 and 3
awk '/Running/{print $1}' file          # filter then print
sed 's/old/new/g' file                  # replace all occurrences
cut -d',' -f2 file.csv                  # cut column by delimiter
sort file | uniq                        # sort and deduplicate
wc -l file                              # count lines
cat file | tr '[:upper:]' '[:lower:]'  # lowercase
```

### Process Management

```bash
ps aux                         # all processes
ps aux | grep kubelet           # find specific process
top                            # live process monitor
htop                           # nicer top (brew install htop)
kill -9 PID                    # force kill
pkill -f "process-name"        # kill by name
systemctl status kubelet        # check service status
systemctl restart kubelet       # restart service
systemctl enable kubelet        # enable on boot
journalctl -u kubelet -f        # follow service logs
journalctl -u kubelet --since "10 min ago"
```

### Networking Basics

```bash
ip addr show                   # network interfaces
ip route show                  # routing table
netstat -tulnp                 # listening ports
ss -tulnp                      # modern netstat
curl -v http://service:8080    # HTTP request with verbose
wget -O /tmp/file http://url   # download file
nslookup kubernetes.default    # DNS lookup
dig kubernetes.default.svc.cluster.local
ping 10.96.0.1                 # connectivity test
traceroute 8.8.8.8             # trace route
iptables -L -n -t nat          # NAT rules (kube-proxy uses these)
```

### SSH & Remote Tools

```bash
ssh user@192.168.1.10                        # basic SSH
ssh -i ~/.ssh/key.pem user@host              # with key
scp file.txt user@host:/remote/path/         # copy to remote
scp user@host:/remote/file.txt ./local/      # copy from remote
ssh -L 8080:localhost:8080 user@host         # port forwarding
```

### YAML Fundamentals

```yaml
# Scalars
name: my-pod
replicas: 3
enabled: true
value: 3.14

# Lists
containers:
  - name: nginx
  - name: redis

# Nested maps
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80

# Multi-line strings
command: |
  echo "line 1"
  echo "line 2"

# Inline list
args: ["--port", "8080"]
```

---

## 1.2 Containers vs VMs

| Feature | VM | Container |
|---------|-----|-----------|
| Isolation | Hardware-level (hypervisor) | OS-level (namespaces + cgroups) |
| Boot time | Minutes | Milliseconds |
| Size | GB | MB |
| OS | Full OS per VM | Shared kernel |
| Use case | Strong isolation needed | Microservices, CI/CD |

### Linux Namespaces (what makes containers)

```
PID namespace    — isolated process IDs
NET namespace    — isolated network stack
MNT namespace    — isolated filesystem mounts
UTS namespace    — isolated hostname
IPC namespace    — isolated inter-process communication
USER namespace   — isolated user IDs
```

### cgroups (resource limits for containers)

```bash
# Docker uses cgroups under the hood
# /sys/fs/cgroup/ contains all limits
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

---

## 1.3 Docker / containerd Basics

### Key Concepts

- **Image** — read-only template (layers)
- **Container** — running instance of an image
- **Registry** — image storage (Docker Hub, ECR, GCR)
- **Dockerfile** — recipe to build an image
- **containerd** — CRI-compatible runtime Kubernetes uses directly

### Essential Docker Commands

```bash
# Images
docker pull nginx:1.25
docker images
docker rmi nginx:1.25
docker build -t myapp:v1 .
docker tag myapp:v1 registry/myapp:v1
docker push registry/myapp:v1

# Containers
docker run -d --name web -p 8080:80 nginx:1.25
docker run -it ubuntu:22.04 bash          # interactive
docker exec -it web bash                  # enter running container
docker logs web -f                        # follow logs
docker inspect web                        # full details JSON
docker stop web
docker rm web
docker ps -a                              # all containers

# Networking
docker network ls
docker network create mynet
docker run -d --network mynet nginx

# Volumes
docker volume create myvol
docker run -v myvol:/data nginx
docker run -v /host/path:/container/path nginx  # bind mount
```

### Dockerfile Example

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY app.py .

EXPOSE 8080

CMD ["python3", "app.py"]
```

### containerd CLI (crictl) — used in CKA

```bash
# On exam nodes, Docker may not be available — use crictl
crictl ps                          # list running containers
crictl images                      # list images
crictl logs <container-id>
crictl inspect <container-id>
crictl pods                        # list pods
crictl pull nginx:1.25
```

---

## Lab 01-A — Linux Essentials

**Goal:** Practice file operations, process inspection, and text processing.

```bash
# 1. Create a workspace
mkdir -p ~/cka-labs/module01 && cd ~/cka-labs/module01

# 2. Create a test file
cat > processes.txt << 'EOF'
nginx   1234  running
kubelet 5678  running
etcd    9012  stopped
redis   3456  running
EOF

# 3. Find all running processes from the file
awk '$3 == "running" {print $1}' processes.txt

# 4. Count lines
wc -l processes.txt

# 5. Sort processes by name
sort -k1 processes.txt

# 6. Replace 'stopped' with 'failed'
sed 's/stopped/failed/g' processes.txt

# 7. Find your current processes related to docker
ps aux | grep docker | awk '{print $1, $2, $11}'

# 8. Check open ports on your Mac
sudo lsof -nP -iTCP -sTCP:LISTEN
```

**Expected output for step 3:**
```
nginx
kubelet
redis
```

---

## Lab 01-B — Docker Hands-On

**Goal:** Build, run, inspect, and debug containers on macOS.

### Step 1 — Start Docker Desktop

Open Docker Desktop from Applications or:
```bash
open -a Docker
# Wait ~30s then verify
docker info
```

### Step 2 — Run your first container

```bash
# Run nginx in detached mode
docker run -d --name my-nginx -p 8080:80 nginx:1.25

# Verify it's running
docker ps

# Test it
curl http://localhost:8080

# Check logs
docker logs my-nginx

# Enter the container
docker exec -it my-nginx bash
  # Inside container:
  cat /etc/nginx/nginx.conf
  ls /usr/share/nginx/html/
  exit
```

### Step 3 — Build a custom image

```bash
mkdir -p ~/cka-labs/module01/myapp && cd ~/cka-labs/module01/myapp

# Create a simple HTML page
cat > index.html << 'EOF'
<html><body><h1>CKA Lab 01 - My Container</h1></body></html>
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:1.25-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF

# Build the image
docker build -t cka-app:v1 .

# Verify image was created
docker images | grep cka-app

# Run your custom image
docker run -d --name cka-app -p 9090:80 cka-app:v1

# Test it
curl http://localhost:9090
```

### Step 4 — Inspect and debug container networking

```bash
# Inspect the container network
docker inspect my-nginx | grep -A 20 '"NetworkSettings"'

# List networks
docker network ls

# Inspect the default bridge network
docker network inspect bridge

# Create a custom network and test DNS
docker network create cka-net

docker run -d --name app1 --network cka-net nginx:alpine
docker run -d --name app2 --network cka-net nginx:alpine

# Test container DNS resolution
docker exec app1 ping -c 3 app2
docker exec app1 curl http://app2

# Cleanup
docker stop my-nginx cka-app app1 app2
docker rm my-nginx cka-app app1 app2
docker network rm cka-net
```

### Step 5 — Volume persistence

```bash
# Create a named volume
docker volume create cka-data

# Run container with volume
docker run -d --name vol-test \
  -v cka-data:/data \
  ubuntu:22.04 \
  bash -c "while true; do date >> /data/log.txt; sleep 1; done"

# Wait 5 seconds then read data
sleep 5
docker exec vol-test cat /data/log.txt

# Stop container, data survives
docker stop vol-test && docker rm vol-test

# New container, same volume — data persists
docker run --rm -v cka-data:/data ubuntu:22.04 cat /data/log.txt

# Cleanup
docker volume rm cka-data
```

---

## Lab 01-C — containerd with nerdctl (Optional)

```bash
# Install nerdctl (containerd CLI)
brew install nerdctl

# Or use Lima (Linux VM on Mac) which bundles containerd
brew install lima

# Start a Linux VM with containerd
limactl start default

# Enter the VM
lima

# Inside Lima VM — use nerdctl same as docker
nerdctl run -d --name test nginx:alpine
nerdctl ps
nerdctl exec -it test sh
nerdctl stop test
nerdctl rm test
```

---

## Checklist

- [ ] Comfortable with `grep`, `awk`, `sed`, `cut`
- [ ] Can inspect processes with `ps` and `systemctl`
- [ ] Can build a Docker image from a Dockerfile
- [ ] Can run/stop/inspect/debug containers
- [ ] Understand container networking (bridge, custom networks)
- [ ] Understand volume persistence
- [ ] Know what `crictl` is and how it differs from `docker`

---

**Next:** [Module 02 — Kubernetes Architecture](./Module-02-Kubernetes-Architecture.md)
