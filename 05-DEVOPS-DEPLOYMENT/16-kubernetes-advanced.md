# Module 16: Kubernetes Advanced

## 📚 Table of Contents
1. [Overview](#overview)
2. [StatefulSets](#statefulsets)
3. [DaemonSets](#daemonsets)
4. [Helm Charts](#helm-charts)
5. [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
6. [Persistent Volumes](#persistent-volumes)
7. [Monitoring and Logging](#monitoring-and-logging)
8. [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
9. [Advanced Patterns](#advanced-patterns)
10. [Best Practices](#best-practices)

---

## Overview

This module covers advanced Kubernetes concepts for production deployments, including StatefulSets for databases, Helm for package management, autoscaling, and monitoring.

**Advanced Topics:**
- StatefulSets for stateful applications
- Helm for application packaging
- Horizontal Pod Autoscaling
- Persistent storage
- Monitoring and observability
- Security with RBAC

---

## StatefulSets

StatefulSets manage stateful applications where Pod identity and ordering matter (e.g., databases).

### Why StatefulSets?

- **Stable network identity**: Each pod gets stable hostname
- **Ordered deployment**: Pods created in order
- **Persistent storage**: Each pod gets its own volume
- **Ordered scaling**: Scale up/down in order

### StatefulSet Example

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**Key Differences from Deployment:**
- Requires `serviceName` (headless service)
- Uses `volumeClaimTemplates` instead of volumes
- Pods named: `postgres-0`, `postgres-1`, `postgres-2`
- Stable DNS: `postgres-0.postgres-headless`, etc.

---

## DaemonSets

DaemonSets ensure a Pod runs on every node (or selected nodes). Used for system-level services.

### DaemonSet Example

```yaml
# daemonset.yaml
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
      - name: log-collector
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**Use Cases:**
- Log collection (fluentd, fluent-bit)
- Monitoring agents (Prometheus node exporter)
- Network proxies
- Storage daemons

---

## Helm Charts

Helm is the package manager for Kubernetes, making it easy to deploy and manage applications.

### Install Helm

```bash
# macOS
brew install helm

# Verify
helm version
```

### Helm Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── charts/            # Chart dependencies
```

### Create a Helm Chart

```bash
# Create new chart
helm create nestjs-app

# Install chart
helm install my-app ./nestjs-app

# Upgrade chart
helm upgrade my-app ./nestjs-app

# Uninstall chart
helm uninstall my-app

# List releases
helm list

# View values
helm get values my-app
```

### Chart.yaml

```yaml
apiVersion: v2
name: nestjs-app
description: A Helm chart for NestJS application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: my-nestjs-app
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nestjs-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "nestjs-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "nestjs-app.name" . }}
    spec:
      containers:
      - name: nestjs
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 3000
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

**Helm Template Functions:**
- `{{ .Values.* }}`: Access values from values.yaml
- `{{ include "helper" . }}`: Include helper templates
- `{{- toYaml }}`: Convert to YAML format

---

## Horizontal Pod Autoscaling

HPA automatically scales the number of Pods based on CPU, memory, or custom metrics.

### Install Metrics Server

```bash
# Apply metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### HPA Example

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nestjs-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nestjs-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**HPA Behavior:**
- Monitors metrics every 15 seconds
- Scales up when metrics exceed target
- Scales down when metrics below target
- Respects min/max replica limits

### Create HPA

```bash
# Create HPA
kubectl apply -f hpa.yaml

# View HPA status
kubectl get hpa

# Describe HPA
kubectl describe hpa nestjs-app-hpa
```

---

## Persistent Volumes

Persistent Volumes (PV) and Persistent Volume Claims (PVC) provide persistent storage.

### Persistent Volume

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/data/postgres
```

### Persistent Volume Claim

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: local-storage
```

### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: postgres
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

---

## Monitoring and Logging

### Prometheus Monitoring

Install Prometheus for metrics collection:

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack
```

### Application Metrics

Expose metrics in your NestJS app:

```typescript
// Install prometheus client
// npm install prom-client

import { Registry, Counter, Histogram } from 'prom-client';

const register = new Registry();

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  registers: [register],
});

// Expose metrics endpoint
@Controller('metrics')
export class MetricsController {
  @Get()
  @Header('Content-Type', register.contentType)
  async getMetrics() {
    return register.metrics();
  }
}
```

### Logging with Fluentd

Deploy Fluentd for log aggregation:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
```

---

## RBAC (Role-Based Access Control)

RBAC controls access to Kubernetes resources.

### Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nestjs-service-account
```

### Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: nestjs-service-account
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Advanced Patterns

### Init Containers

Init containers run before main containers:

```yaml
spec:
  initContainers:
  - name: init-db
    image: postgres:14-alpine
    command: ['sh', '-c', 'until pg_isready -h postgres; do sleep 2; done']
  containers:
  - name: nestjs
    image: my-nestjs-app:latest
```

### Sidecar Containers

Sidecars run alongside main containers:

```yaml
spec:
  containers:
  - name: nestjs
    image: my-nestjs-app:latest
  - name: log-sidecar
    image: fluent/fluent-bit:latest
```

### Jobs and CronJobs

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: my-nestjs-app:latest
        command: ["npm", "run", "migration:run"]
      restartPolicy: Never
```

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: my-nestjs-app:latest
            command: ["npm", "run", "backup"]
          restartPolicy: OnFailure
```

---

## Best Practices

### 1. Use Resource Limits

Always set resource requests and limits.

### 2. Use Health Checks

Implement liveness and readiness probes.

### 3. Use ConfigMaps and Secrets

Never hardcode configuration or secrets.

### 4. Use Namespaces

Organize resources with namespaces.

### 5. Use Helm for Deployments

Package applications with Helm charts.

### 6. Monitor Applications

Set up monitoring and alerting.

### 7. Use HPA for Auto-scaling

Automatically scale based on load.

---

## Next Steps

✅ Understand StatefulSets and DaemonSets  
✅ Use Helm for application management  
✅ Implement autoscaling  
✅ Set up monitoring  
✅ Configure RBAC  
✅ Move to Module 17: CI/CD Pipeline  

---

*You now know advanced Kubernetes concepts! ☸️*

