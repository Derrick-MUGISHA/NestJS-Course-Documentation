# Module 24: Production Deployment

## 📚 Table of Contents
1. [Overview](#overview)
2. [Production Checklist](#production-checklist)
3. [Environment Configuration](#environment-configuration)
4. [Database Migrations](#database-migrations)
5. [Deployment Strategies](#deployment-strategies)
6. [Blue-Green Deployment](#blue-green-deployment)
7. [Canary Deployment](#canary-deployment)
8. [Rollback Strategies](#rollback-strategies)
9. [Backup and Recovery](#backup-and-recovery)
10. [Disaster Recovery](#disaster-recovery)
11. [Monitoring Production](#monitoring-production)
12. [Scaling Production](#scaling-production)
13. [Best Practices](#best-practices)

---

## Overview

This final module covers deploying NestJS applications to production, including deployment strategies, monitoring, scaling, and disaster recovery.

**Topics:**
- Production deployment checklist
- Deployment strategies
- Rollback procedures
- Backup and recovery
- Production monitoring
- Scaling strategies

---

## Production Checklist

### Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Environment variables configured
- [ ] Database migrations ready
- [ ] Secrets properly managed
- [ ] Logging configured
- [ ] Monitoring set up
- [ ] Health checks implemented
- [ ] Rate limiting enabled
- [ ] CORS configured
- [ ] HTTPS enabled
- [ ] Error tracking configured
- [ ] Backup strategy in place

---

## Environment Configuration

### Production Environment Variables

```env
# Production .env
NODE_ENV=production
PORT=3000

# Database
DATABASE_HOST=production-db.example.com
DATABASE_PORT=5432
DATABASE_USERNAME=app_user
DATABASE_PASSWORD=${DB_PASSWORD}  # From secret manager
DATABASE_NAME=production_db

# Security
JWT_SECRET=${JWT_SECRET}  # From secret manager
JWT_EXPIRATION=1h

# Redis
REDIS_HOST=production-redis.example.com
REDIS_PORT=6379
REDIS_PASSWORD=${REDIS_PASSWORD}

# CORS
FRONTEND_URL=https://app.example.com

# Monitoring
SENTRY_DSN=${SENTRY_DSN}
```

### Use Secret Managers

**AWS Secrets Manager:**
```typescript
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'app-secrets' });
```

**Kubernetes Secrets:**
```bash
kubectl create secret generic app-secrets \
  --from-literal=database-password=secret \
  --from-literal=jwt-secret=secret
```

---

## Database Migrations

### Run Migrations in Production

```typescript
// migration.service.ts
@Injectable()
export class MigrationService {
  constructor(private dataSource: DataSource) {}

  async runMigrations() {
    try {
      const migrations = await this.dataSource.runMigrations();
      console.log(`Ran ${migrations.length} migrations`);
      return migrations;
    } catch (error) {
      console.error('Migration failed:', error);
      throw error;
    }
  }
}

// Run on startup
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Run migrations
  const migrationService = app.get(MigrationService);
  await migrationService.runMigrations();
  
  await app.listen(3000);
}
```

### Migration Best Practices

1. **Test migrations on staging first**
2. **Backup database before migrations**
3. **Make migrations idempotent**
4. **Keep migrations small**
5. **Never modify existing migrations**

---

## Deployment Strategies

### Rolling Update (Default Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Can have 1 extra pod during update
      maxUnavailable: 0  # Always have at least 3 pods available
```

**How it works:**
- Creates new pod with new version
- Waits for new pod to be ready
- Terminates old pod
- Repeats until all pods updated

### Recreate Strategy

```yaml
strategy:
  type: Recreate  # Deletes all old pods, then creates new ones
```

**Use when:** Application doesn't support multiple versions simultaneously.

---

## Blue-Green Deployment

Blue-Green deployment runs two identical production environments.

### Blue-Green with Kubernetes

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  template:
    metadata:
      labels:
        version: blue
    spec:
      containers:
      - name: app
        image: my-app:v1.0.0

---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        version: green
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0

---
# Service (switch between blue/green)
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    version: blue  # Switch to 'green' for new version
  ports:
  - port: 80
```

**Blue-Green Process:**
1. Deploy green version alongside blue
2. Test green version
3. Switch traffic to green
4. Keep blue for quick rollback
5. Delete blue after verification

---

## Canary Deployment

Canary deployment gradually shifts traffic to new version.

### Canary with Kubernetes

```yaml
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  template:
    metadata:
      labels:
        version: stable
    spec:
      containers:
      - name: app
        image: my-app:v1.0.0

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  template:
    metadata:
      labels:
        version: canary
    spec:
      containers:
      - name: app
        image: my-app:v2.0.0

---
# Service with traffic splitting
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: my-app  # Routes to both stable and canary
  ports:
  - port: 80
```

**Canary Process:**
1. Deploy canary version (small % of traffic)
2. Monitor metrics
3. Gradually increase canary traffic
4. Rollback if issues detected
5. Promote to stable if successful

---

## Rollback Strategies

### Kubernetes Rollback

```bash
# Rollback deployment
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# View rollout history
kubectl rollout history deployment/my-app

# View specific revision
kubectl rollout history deployment/my-app --revision=2
```

### Application-Level Rollback

```typescript
// Keep old version running
// Switch load balancer back to old version
// Monitor for issues
// Delete new version if rollback successful
```

---

## Backup and Recovery

### Database Backups

```bash
# Automated backup script
#!/bin/bash
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).sql"
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME > $BACKUP_FILE
gzip $BACKUP_FILE
aws s3 cp $BACKUP_FILE.gz s3://backups/database/
```

### Scheduled Backups

```yaml
# Kubernetes CronJob for backups
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command: ["/bin/bash", "-c", "pg_dump ..."]
          restartPolicy: OnFailure
```

---

## Disaster Recovery

### Recovery Procedures

1. **Identify failure point**
2. **Assess impact**
3. **Activate backup systems**
4. **Restore from backups**
5. **Verify data integrity**
6. **Resume operations**

### Recovery Time Objectives (RTO)

- **RTO**: Maximum acceptable downtime
- **RPO**: Maximum acceptable data loss

Plan for:
- Database failures
- Application failures
- Infrastructure failures
- Region failures

---

## Monitoring Production

### Key Metrics to Monitor

1. **Application Metrics**
   - Request rate
   - Error rate
   - Response time
   - CPU/Memory usage

2. **Database Metrics**
   - Connection pool usage
   - Query performance
   - Replication lag

3. **Infrastructure Metrics**
   - Server resources
   - Network latency
   - Disk usage

### Alerting Rules

Set up alerts for:
- High error rates (>5%)
- Slow response times (>2s)
- High resource usage (>80%)
- Service downtime

---

## Scaling Production

### Horizontal Scaling

```bash
# Scale deployment
kubectl scale deployment/my-app --replicas=10

# Auto-scaling
kubectl autoscale deployment/my-app --min=3 --max=20 --cpu-percent=70
```

### Vertical Scaling

Increase resource limits:

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

---

## Best Practices

### 1. Use Infrastructure as Code

Version control your infrastructure (Terraform, CloudFormation).

### 2. Implement Health Checks

Monitor application health continuously.

### 3. Use Feature Flags

Enable/disable features without deployment.

### 4. Monitor Everything

Logs, metrics, traces - monitor all.

### 5. Automate Deployments

Use CI/CD pipelines for consistency.

### 6. Test Rollback Procedures

Regularly test disaster recovery.

### 7. Document Runbooks

Document procedures for common issues.

### 8. Regular Backups

Automate and test backups.

---

## Congratulations! 🎉

You've completed the comprehensive NestJS training program! You now have the knowledge to:

✅ Build scalable NestJS applications  
✅ Design complex database schemas  
✅ Implement microservices  
✅ Deploy to production  
✅ Monitor and optimize applications  
✅ Handle distributed systems  
✅ Secure applications  
✅ Scale horizontally  

---

*You are now a NestJS Hero! 🚀*

