# Module 22: Observability & Monitoring

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is Observability?](#what-is-observability)
3. [Structured Logging](#structured-logging)
4. [Prometheus Metrics](#prometheus-metrics)
5. [Distributed Tracing](#distributed-tracing)
6. [Health Checks](#health-checks)
7. [Error Tracking](#error-tracking)
8. [APM Tools](#apm-tools)
9. [Alerting](#alerting)
10. [Best Practices](#best-practices)

---

## Overview

Observability is the ability to understand system behavior through logs, metrics, and traces. This module covers implementing comprehensive observability in NestJS applications.

**Three Pillars of Observability:**
1. **Logs**: Events that happened
2. **Metrics**: Measurements over time
3. **Traces**: Request flows through services

**Benefits:**
- Debug issues faster
- Understand system performance
- Predict problems
- Optimize performance

---

## What is Observability?

Observability = Logs + Metrics + Traces

**Logs**: What happened and when
**Metrics**: How much, how fast
**Traces**: Where and why

---

## Structured Logging

Structured logs are machine-readable and easier to parse.

### Install Winston

```bash
npm install nest-winston winston
npm install winston-daily-rotate-file
```

### Configure Winston

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { WINSTON_MODULE_NEST_PROVIDER } from 'nest-winston';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Use Winston logger
  app.useLogger(app.get(WINSTON_MODULE_NEST_PROVIDER));

  await app.listen(3000);
}
bootstrap();
```

### Winston Configuration

```typescript
// winston.config.ts
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';
import * as DailyRotateFile from 'winston-daily-rotate-file';

export const winstonConfig = WinstonModule.createLogger({
  transports: [
    // Console transport
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.colorize(),
        winston.format.printf(({ timestamp, level, message, context, ...meta }) => {
          return `${timestamp} [${context}] ${level}: ${message} ${Object.keys(meta).length ? JSON.stringify(meta) : ''}`;
        }),
      ),
    }),

    // File transport - all logs
    new DailyRotateFile({
      filename: 'logs/application-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      zippedArchive: true,
      maxSize: '20m',
      maxFiles: '14d',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(), // Structured JSON format
      ),
    }),

    // File transport - errors only
    new DailyRotateFile({
      filename: 'logs/error-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      level: 'error',
      zippedArchive: true,
      maxSize: '20m',
      maxFiles: '30d',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
  ],
});
```

### Using Logger

```typescript
// users.service.ts
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async create(createUserDto: CreateUserDto): Promise<User> {
    // Log with context
    this.logger.log(`Creating user with email: ${createUserDto.email}`, {
      email: createUserDto.email,
      timestamp: new Date().toISOString(),
    });

    try {
      const user = await this.usersRepository.save(createUserDto);

      this.logger.log(`User created successfully: ${user.id}`, {
        userId: user.id,
        email: user.email,
      });

      return user;
    } catch (error) {
      // Log errors with stack trace
      this.logger.error(`Failed to create user: ${error.message}`, {
        error: error.message,
        stack: error.stack,
        email: createUserDto.email,
      });
      throw error;
    }
  }
}
```

### Structured Logging Best Practices

```typescript
// Good: Structured log with context
this.logger.log('User logged in', {
  userId: user.id,
  email: user.email,
  ip: request.ip,
  userAgent: request.headers['user-agent'],
  timestamp: new Date().toISOString(),
});

// Bad: Unstructured log
console.log('User logged in'); // ❌
```

---

## Prometheus Metrics

Prometheus collects and stores metrics.

### Install Prometheus Client

```bash
npm install prom-client
```

### Metrics Setup

```typescript
// metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Registry, Counter, Histogram, Gauge } from 'prom-client';

@Injectable()
export class MetricsService {
  private readonly register: Registry;

  // Counter: Total number of requests
  private readonly httpRequestCounter: Counter;
  
  // Histogram: Request duration
  private readonly httpRequestDuration: Histogram;
  
  // Gauge: Current active connections
  private readonly activeConnections: Gauge;

  constructor() {
    this.register = new Registry();

    // Counter for HTTP requests
    this.httpRequestCounter = new Counter({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status'],
      registers: [this.register],
    });

    // Histogram for request duration
    this.httpRequestDuration = new Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status'],
      buckets: [0.1, 0.5, 1, 2, 5], // Duration buckets
      registers: [this.register],
    });

    // Gauge for active connections
    this.activeConnections = new Gauge({
      name: 'active_connections',
      help: 'Number of active connections',
      registers: [this.register],
    });
  }

  // Record HTTP request
  recordHttpRequest(method: string, route: string, statusCode: number, duration: number) {
    this.httpRequestCounter.inc({
      method,
      route,
      status: statusCode.toString(),
    });

    this.httpRequestDuration.observe(
      {
        method,
        route,
        status: statusCode.toString(),
      },
      duration,
    );
  }

  // Update active connections
  setActiveConnections(count: number) {
    this.activeConnections.set(count);
  }

  // Get metrics for Prometheus
  async getMetrics(): Promise<string> {
    return this.register.metrics();
  }
}
```

### Metrics Controller

```typescript
// metrics.controller.ts
import { Controller, Get, Header } from '@nestjs/common';
import { MetricsService } from './metrics.service';

@Controller('metrics')
export class MetricsController {
  constructor(private metricsService: MetricsService) {}

  @Get()
  @Header('Content-Type', 'text/plain')
  async getMetrics() {
    return this.metricsService.getMetrics();
  }
}
```

### Metrics Interceptor

```typescript
// metrics.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { MetricsService } from './metrics.service';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(private metricsService: MetricsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, route } = request;
    const startTime = Date.now();

    return next.handle().pipe(
      tap({
        next: (response) => {
          const duration = (Date.now() - startTime) / 1000;
          const statusCode = response?.statusCode || 200;
          const routePath = route?.path || request.url;

          this.metricsService.recordHttpRequest(
            method,
            routePath,
            statusCode,
            duration,
          );
        },
        error: (error) => {
          const duration = (Date.now() - startTime) / 1000;
          const statusCode = error.status || 500;
          const routePath = route?.path || request.url;

          this.metricsService.recordHttpRequest(
            method,
            routePath,
            statusCode,
            duration,
          );
        },
      }),
    );
  }
}
```

---

## Distributed Tracing

Distributed tracing tracks requests across multiple services.

### OpenTelemetry Setup

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node
npm install @opentelemetry/instrumentation-http @opentelemetry/instrumentation-express
```

### Tracing Configuration

```typescript
// tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';

const sdk = new NodeSDK({
  traceExporter: new JaegerExporter({
    endpoint: 'http://localhost:14268/api/traces',
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

sdk.start();
```

### Custom Spans

```typescript
// users.service.ts
import { trace, Span } from '@opentelemetry/api';

@Injectable()
export class UsersService {
  private tracer = trace.getTracer('users-service');

  async create(createUserDto: CreateUserDto): Promise<User> {
    // Create span for this operation
    const span = this.tracer.startSpan('users.create');

    try {
      span.setAttributes({
        'user.email': createUserDto.email,
        'operation': 'create_user',
      });

      const user = await this.usersRepository.save(createUserDto);

      span.setAttribute('user.id', user.id);
      span.setStatus({ code: 1 }); // OK

      return user;
    } catch (error) {
      span.setStatus({
        code: 2, // ERROR
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }
}
```

---

## Health Checks

Health checks monitor service availability.

### Health Check Module

```bash
npm install @nestjs/terminus
```

### Health Check Setup

```typescript
// health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator,
} from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      // Database health
      () => this.db.pingCheck('database'),

      // Memory health
      () =>
        this.memory.checkHeap('memory_heap', 150 * 1024 * 1024), // 150MB
      () =>
        this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),

      // Disk health
      () =>
        this.disk.checkStorage('storage', {
          path: '/',
          thresholdPercent: 0.9, // 90% full
        }),
    ]);
  }

  @Get('readiness')
  @HealthCheck()
  readiness() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }

  @Get('liveness')
  @HealthCheck()
  liveness() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
```

---

## Error Tracking

### Sentry Integration

```bash
npm install @sentry/node @sentry/tracing
```

```typescript
// main.ts
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
  profilesSampleRate: 1.0,
  integrations: [nodeProfilingIntegration()],
});

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

---

## APM Tools

### New Relic

```typescript
// Install newrelic package
require('newrelic'); // Must be first require

// newrelic.js config file
module.exports = {
  app_name: ['My NestJS App'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  distributed_tracing: {
    enabled: true,
  },
};
```

---

## Alerting

### Alert Rules (Prometheus)

```yaml
# alerts.yml
groups:
  - name: application_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} requests/second"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, http_request_duration_seconds) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
```

---

## Best Practices

### 1. Use Structured Logging

Always use structured, machine-readable logs.

### 2. Include Context

Add relevant context to all logs (userId, requestId, etc.).

### 3. Set Appropriate Log Levels

- ERROR: Errors requiring attention
- WARN: Warnings
- INFO: Important information
- DEBUG: Debug information
- VERBOSE: Very detailed

### 4. Sample Traces

Don't trace 100% of requests in production.

### 5. Monitor Key Metrics

- Request rate
- Error rate
- Response time
- Resource usage

---

## Next Steps

✅ Implement structured logging  
✅ Set up Prometheus metrics  
✅ Configure distributed tracing  
✅ Add health checks  
✅ Set up error tracking  
✅ Move to Module 23: Security & Performance  

---

*You now know how to observe and monitor your applications! 📊*

