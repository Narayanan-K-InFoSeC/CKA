# CKA 2026 — Complete Study Guide

> Certified Kubernetes Administrator — Full course with theory + hands-on labs for macOS

---

## Prerequisites — Local Setup (macOS)

### Required Tools

```bash
# Install Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Docker Desktop
brew install --cask docker

# Install kubectl
brew install kubectl

# Install minikube (primary local cluster)
brew install minikube

# Install kind (Kubernetes in Docker) — alternative
brew install kind

# Install Helm
brew install helm

# Install k9s (optional but highly recommended TUI)
brew install k9s

# Verify installations
docker --version
kubectl version --client
minikube version
helm version
```

### Start Your Local Cluster

```bash
# Start minikube with enough resources
minikube start --cpus=4 --memory=8192 --driver=docker

# Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

---

## Course Modules

| Module | Topic | Weight |
|--------|-------|--------|
| [00](./Module-00-Local-Setup-macOS.md) | Local Setup — macOS Practice Environment | Prerequisites |
| [01](./Module-01-Linux-Container-Fundamentals.md) | Linux & Container Fundamentals | Foundation |
| [02](./Module-02-Kubernetes-Architecture.md) | Kubernetes Architecture | ~10% |
| [03](./Module-03-Cluster-Installation-Configuration.md) | Cluster Installation & Configuration | ~15% |
| [04](./Module-04-Kubernetes-Core-Objects.md) | Kubernetes Core Objects | ~20% |
| [05](./Module-05-Scheduling.md) | Scheduling | ~10% |
| [06](./Module-06-Services-Networking.md) | Services & Networking | ~20% |
| [07](./Module-07-Storage.md) | Storage | ~10% |
| [08](./Module-08-Security.md) | Security | ~10% |
| [09](./Module-09-Logging-Monitoring.md) | Logging & Monitoring | ~5% |
| [10](./Module-10-Troubleshooting.md) | Troubleshooting | **HIGHEST** |
| [11](./Module-11-Helm-Package-Management.md) | Helm & Package Management | ~5% |
| [12](./Module-12-Exam-Preparation.md) | Exam Preparation | — |

---

## Exam Quick Facts (2026)

- **Duration:** 2 hours
- **Format:** Performance-based (live terminal in browser)
- **Pass Score:** 66%
- **Allowed:** kubernetes.io/docs (one tab)
- **Version:** Kubernetes 1.31+

---

## Daily Study Routine

```
Day 0     → Module 00 (Install all tools — do this before anything else)
Day 1–2   → Module 01 (Linux + Docker)
Day 3–4   → Module 02 (Architecture)
Day 5–7   → Module 03 (Cluster setup — kubeadm)
Day 8–10  → Module 04 (Core objects)
Day 11–12 → Module 05 (Scheduling)
Day 13–15 → Module 06 (Networking) ← heavy
Day 16–17 → Module 07 (Storage)
Day 18–19 → Module 08 (Security)
Day 20    → Module 09 (Logging)
Day 21–25 → Module 10 (Troubleshooting) ← heaviest
Day 26    → Module 11 (Helm)
Day 27–30 → Module 12 (Mock exams + speed drills)
```
