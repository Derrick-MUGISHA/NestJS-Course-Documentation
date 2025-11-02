# Module 15: Kubernetes Basics

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is Kubernetes?](#what-is-kubernetes)
3. [Kubernetes Installation](#kubernetes-installation)
4. [Core Concepts](#core-concepts)
5. [Pods](#pods)
6. [Deployments](#deployments)
7. [Services](#services)
8. [ConfigMaps and Secrets](#configmaps-and-secrets)
9. [Ingress](#ingress)
10. [Resource Management](#resource-management)
11. [Basic Commands](#basic-commands)
12. [Mini Project: Deploy NestJS App](#mini-project-deploy-nestjs-app)

---

## Overview

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications. This module introduces Kubernetes fundamentals for deploying NestJS applications.

**Why Kubernetes?**
- **Auto-scaling**: Automatically scale based on load
- **Self-healing**: Restart failed containers automatically
- **Load Balancing**: Distribute traffic across pods
- **Rolling Updates**: Update applications with zero downtime
- **Resource Management**: Efficiently use cluster resources

**Kubernetes vs Docker Compose:**
- Docker Compose: Single machine, local development
- Kubernetes: Multiple machines, production orchestration

---

## What is Kubernetes?

Kubernetes orchestrates containers across a cluster of machines. It handles:
- Scheduling containers on nodes
- Managing container lifecycle
- Service discovery and load balancing
- Storage orchestration
- Health monitoring and self-healing

### Architecture

**Master Node (Control Plane):**
- API Server: Entry point for all operations
- etcd: Cluster state storage
- Scheduler: Assigns pods to nodes
- Controller Manager: Manages controllers

**Worker Nodes:**
- kubelet: Container runtime interface
- kube-proxy: Network proxy
- Container runtime: Docker, containerd, etc.

---

## Kubernetes Installation

### Local Development: Minikube

Minikube runs a single-node Kubernetes cluster locally:

```bash
# macOS
brew install minikube

# Start Minikube
minikube start

# Check status
minikube status

# Stop Minikube
minikube stop
```

### Local Development: kind (Kubernetes in Docker)

```bash
# Install kind
brew install kind

# Create cluster
kind create cluster --name my-cluster

# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name my-cluster
```

### kubectl Installation

kubectl is the Kubernetes command-line tool:

```bash
# macOS
brew install kubectl

# Verify installation
kubectl version --client

# Check cluster connection
kubectl cluster-info
```

---

## Core Concepts

### Pods

A **Pod** is the smallest deployable unit in Kubernetes. It contains one or more containers that share storage and network.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nestjs-app
  labels:
    app: nestjs-app
spec:
  containers:
  - name: nestjs
    image: my-nestjs-app:latest
    ports:
    - containerPort: 3000
    env:
    - name: NODE_ENV
      value: "production"
```

**Key Pod Characteristics:**
- Ephemeral: Can be created and destroyed
- Shared network: Containers in same pod share IP
- Shared storage: Containers can share volumes

### Deployments

A **Deployment** manages Pod replicas and provides declarative updates.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
  labels:
    app: nestjs-app
spec:
  # Number of replica pods
  replicas: 3
  
  # Pod selector (which pods belong to this deployment)
  selector:
    matchLabels:
      app: nestjs-app
  
  # Pod template
  template:
    metadata:
      labels:
        app: nestjs-app
    spec:
      containers:
      - name: nestjs
        image: my-nestjs-app:latest
        ports:
        - containerPort: 3000
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Environment variables
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Deployment Benefits:**
- Replica management: Maintains desired number of pods
- Rolling updates: Updates pods without downtime
- Rollback: Revert to previous version
- Self-healing: Replaces failed pods

### Services

A **Service** exposes Pods to network traffic (internal or external).

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nestjs-service
spec:
  # Service type: ClusterIP (internal), NodePort, LoadBalancer
  type: ClusterIP
  
  # Selector: Which pods this service targets
  selector:
    app: nestjs-app
  
  # Port configuration
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 3000  # Pod port
```

**Service Types:**
- **ClusterIP**: Internal access only (default)
- **NodePort**: Expose on each node's IP
- **LoadBalancer**: External load balancer (cloud providers)
- **ExternalName**: Maps to external DNS name

---

## ConfigMaps and Secrets

### ConfigMaps

ConfigMaps store non-sensitive configuration data:

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
```

**Using ConfigMap in Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: nestjs
        envFrom:
        - configMapRef:
            name: app-config
```

### Secrets

Secrets store sensitive data (passwords, API keys):

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # Base64 encoded values
  database-password: cG9zdGdyZXM=  # "postgres" in base64
  jwt-secret: c2VjcmV0LWtleQ==     # "secret-key" in base64
```

**Create Secret from file:**

```bash
# Create secret from literal
kubectl create secret generic app-secrets \
  --from-literal=database-password=postgres \
  --from-literal=jwt-secret=secret-key

# Create secret from file
kubectl create secret generic app-secrets \
  --from-file=.env
```

**Using Secret in Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: nestjs
        envFrom:
        - secretRef:
            name: app-secrets
```

---

## Ingress

Ingress manages external HTTP/HTTPS access to services.

### Install Ingress Controller

```bash
# Using Minikube
minikube addons enable ingress

# Or install NGINX Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### Ingress Resource

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nestjs-ingress
  annotations:
    # NGINX Ingress annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  # Host-based routing
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nestjs-service
            port:
              number: 80
  
  # TLS/SSL configuration
  tls:
  - hosts:
    - api.myapp.com
    secretName: tls-secret
```

---

## Resource Management

### Resource Requests and Limits

```yaml
spec:
  containers:
  - name: nestjs
    resources:
      # Requests: Minimum resources guaranteed
      requests:
        memory: "256Mi"
        cpu: "250m"  # 0.25 CPU cores
      
      # Limits: Maximum resources allowed
      limits:
        memory: "512Mi"
        cpu: "500m"  # 0.5 CPU cores
```

**Resource Units:**
- CPU: `1000m` = 1 core, `500m` = 0.5 cores
- Memory: `Mi` (Mebibytes), `Gi` (Gibibytes)

---

## Basic Commands

### Apply Resources

```bash
# Apply YAML file
kubectl apply -f deployment.yaml

# Apply all YAML files in directory
kubectl apply -f k8s/

# Create resource from YAML
kubectl create -f deployment.yaml
```

### View Resources

```bash
# List pods
kubectl get pods

# List with more details
kubectl get pods -o wide

# Describe a pod
kubectl describe pod <pod-name>

# List deployments
kubectl get deployments

# List services
kubectl get services

# List all resources
kubectl get all
```

### Logs and Debugging

```bash
# View pod logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>

# Logs from all pods in deployment
kubectl logs -f deployment/nestjs-app

# Execute command in pod
kubectl exec -it <pod-name> -- sh

# Get pod shell
kubectl exec -it <pod-name> -- /bin/bash
```

### Scaling

```bash
# Scale deployment
kubectl scale deployment nestjs-app --replicas=5

# Scale based on CPU (requires metrics server)
kubectl autoscale deployment nestjs-app --min=2 --max=10 --cpu-percent=80
```

### Updates and Rollbacks

```bash
# Update image
kubectl set image deployment/nestjs-app nestjs=my-nestjs-app:v2.0.0

# Rollback to previous version
kubectl rollout undo deployment/nestjs-app

# View rollout history
kubectl rollout history deployment/nestjs-app

# Rollback to specific revision
kubectl rollout undo deployment/nestjs-app --to-revision=2

# Check rollout status
kubectl rollout status deployment/nestjs-app
```

### Delete Resources

```bash
# Delete deployment
kubectl delete deployment nestjs-app

# Delete by file
kubectl delete -f deployment.yaml

# Delete all resources
kubectl delete all --all
```

---

## Mini Project: Deploy NestJS App

Deploy a complete NestJS application to Kubernetes:

1. **Create Deployment** with 3 replicas
2. **Create Service** for internal access
3. **Create ConfigMap** for configuration
4. **Create Secret** for sensitive data
5. **Create Ingress** for external access

### Complete Deployment Files

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nestjs-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nestjs-app
  template:
    metadata:
      labels:
        app: nestjs-app
    spec:
      containers:
      - name: nestjs
        image: my-nestjs-app:latest
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nestjs-service
spec:
  selector:
    app: nestjs-app
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
```

**configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  PORT: "3000"
```

**secret.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  database-password: postgres
  jwt-secret: my-secret-key
```

**ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nestjs-ingress
spec:
  rules:
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nestjs-service
            port:
              number: 80
```

### Deploy Steps

```bash
# 1. Apply all resources
kubectl apply -f k8s/

# 2. Check deployment status
kubectl get deployments
kubectl get pods

# 3. Check service
kubectl get services

# 4. Check ingress
kubectl get ingress

# 5. View logs
kubectl logs -f deployment/nestjs-app

# 6. Scale if needed
kubectl scale deployment nestjs-app --replicas=5
```

---

## Next Steps

✅ Understand Kubernetes concepts  
✅ Create Deployments and Services  
✅ Use ConfigMaps and Secrets  
✅ Configure Ingress  
✅ Manage resources  
✅ Move to Module 16: Kubernetes Advanced  

---

*You now know the fundamentals of Kubernetes! ☸️*

