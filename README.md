# n8n on Kubernetes - Deployment Guide

This repository provides a step-by-step guide for deploying [n8n](https://n8n.io), a workflow automation tool, on Kubernetes using PostgreSQL as the database.

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Repository Structure](#repository-structure)
* [Prerequisites](#prerequisites)
* [Quick Start](#quick-start)
* [Configuration](#configuration)
* [Monitoring](#monitoring)
* [Troubleshooting](#troubleshooting)
* [Cleanup](#cleanup)
* [License](#license)
* [Website & Blog](#website--blog)

---

## Overview

This guide sets up a production-ready n8n instance on Kubernetes, featuring:

* Kubernetes deployment with scaling
* SSL/TLS with cert-manager
* PostgreSQL as persistent data store
* Persistent volumes for long-term storage

---

## Architecture

```
┌─────────────┐     ┌──────────────┐
│  Route 53   │───▶│  LoadBalancer │
│   (DNS)     │     │ (Nginx+SSL)  │
└─────────────┘     └──────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │  EKS Cluster│
                   ├─────────────┤
                   │    n8n      │
                   │ PostgreSQL  │
                   └─────────────┘
```

---

## Repository Structure

```
n8n/
├── kubernetes/
│   ├── n8n/
│   │   ├── n8n-claim0-persistentvolumeclaim.yaml
│   │   ├── n8n-deployment.yaml
│   │   └── n8n-service.yaml
│   └── postgres/
│       ├── postgres-claim0-persistentvolumeclaim.yaml
│       ├── postgres-configmap.yaml
│       ├── postgres-deployment.yaml
│       └── postgres-service.yaml
├── workflows/
└── README.md
```

---

## Prerequisites

Install the following tools:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# jq
sudo apt-get update && sudo apt-get install -y jq
```

Verify installations:

```bash
command -v kubectl && command -v helm && command -v jq
```

Export your domain:

```bash
export YOUR_DOMAIN="example.com"
```

---

## Quick Start

### 1. Configure n8n manifests

```bash
cp kubernetes/n8n/n8n-service.yaml kubernetes/n8n/n8n-service-configured.yaml
cp kubernetes/n8n/n8n-deployment.yaml kubernetes/n8n/n8n-deployment-configured.yaml
```

Replace `<YOUR_DOMAIN>` with your actual domain:

Linux:

```bash
sed -i "s/<YOUR_DOMAIN>/$YOUR_DOMAIN/g" kubernetes/n8n/n8n-deployment-configured.yaml
sed -i "s/<YOUR_DOMAIN>/$YOUR_DOMAIN/g" kubernetes/n8n/n8n-service-configured.yaml
```

macOS:

```bash
sed -i '' "s|<YOUR_DOMAIN>|$YOUR_DOMAIN|g" kubernetes/n8n/n8n-deployment-configured.yaml
sed -i '' "s|<YOUR_DOMAIN>|$YOUR_DOMAIN|g" kubernetes/n8n/n8n-service-configured.yaml
```

### 2. Create Namespace & Secret

```bash
kubectl create namespace n8n

kubectl create secret generic postgres-secret \
  --namespace=n8n \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=$(openssl rand -base64 20) \
  --from-literal=POSTGRES_DB=n8n \
  --from-literal=POSTGRES_NON_ROOT_USER=n8n \
  --from-literal=POSTGRES_NON_ROOT_PASSWORD=$(openssl rand -base64 20)
```

### 3. Deploy n8n Stack

```bash
kubectl apply -f kubernetes/postgres/
kubectl apply -f kubernetes/n8n/n8n-claim0-persistentvolumeclaim.yaml
kubectl apply -f kubernetes/n8n/n8n-deployment-configured.yaml
kubectl apply -f kubernetes/n8n/n8n-service-configured.yaml
```

Wait for pods:

```bash
kubectl wait --for=condition=ready pod -l service=postgres-n8n -n n8n --timeout=120s
kubectl wait --for=condition=ready pod -l service=n8n -n n8n --timeout=120s
```

---

## Configuration

| Variable               | Description           |
| ---------------------- | --------------------- |
| N8N\_PROTOCOL          | http or https         |
| N8N\_PORT              | Port (default 5678)   |
| N8N\_EDITOR\_BASE\_URL | Public domain URL     |
| N8N\_RUNNERS\_ENABLED  | Enable runners (true) |

---

## Monitoring

```bash
kubectl get pods -n n8n
kubectl logs -n n8n deployment/n8n
kubectl logs -n n8n deployment/postgres
```

To scale:

```bash
kubectl scale deployment n8n -n n8n --replicas=3
```

---

## Troubleshooting

1. Pod not starting:

   ```bash
   kubectl describe pod -n n8n <pod-name>
   ```

2. Check persistent volume:

   ```bash
   kubectl get pvc -n n8n
   ```

3. Debug events:

   ```bash
   kubectl get events -n n8n --sort-by=.metadata.creationTimestamp
   ```

---

## Cleanup

```bash
kubectl delete namespace n8n
```

---

## License

MIT or Public Domain (check LICENSE file)

---

## Website & Blog

* 🌐 Main Website: [surajkumarsah.com.np](https://surajkumarsah.com.np/)
* ✍️ Technical Blog: [blog.surajkumarsah.com.np](https://blog.surajkumarsah.com.np/)
