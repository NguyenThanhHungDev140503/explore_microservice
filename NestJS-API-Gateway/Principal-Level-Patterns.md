# Principal-Level Patterns: NestJS API Gateway

> **Dành cho:** Principal/Staff Engineer  
> **Nội dung:** Enterprise architecture, multi-region, migration strategies, observability

---

## Mục Lục

1. [Enterprise Architecture](#1-enterprise-architecture)
2. [Multi-Region Deployment](#2-multi-region-deployment)
3. [API Versioning Strategy](#3-api-versioning-strategy)
4. [Migration Strategies](#4-migration-strategies)
5. [Production Observability](#5-production-observability)

---

## 1. Enterprise Architecture

### 1.1. Backend for Frontend (BFF) Pattern

Tạo gateway riêng cho từng loại client:

```
┌─────────────┐     ┌─────────────────┐
│ Mobile App  │────▶│  Mobile BFF     │────┐
└─────────────┘     └─────────────────┘    │
                                           │
┌─────────────┐     ┌─────────────────┐    │     ┌─────────────────┐
│ Web App     │────▶│  Web BFF        │────┼────▶│ Microservices   │
└─────────────┘     └─────────────────┘    │     └─────────────────┘
                                           │
┌─────────────┐     ┌─────────────────┐    │
│ Partner API │────▶│  Partner BFF    │────┘
└─────────────┘     └─────────────────┘
```

### 1.2. Monorepo Structure với Nx

```bash
# Nx workspace structure
apps/
├── mobile-bff/      # Optimized cho mobile
├── web-bff/         # Optimized cho web
├── partner-bff/     # External API
├── user-service/
├── order-service/
└── payment-service/

libs/
├── shared/
│   ├── auth/        # Shared JWT strategy
│   ├── logging/     # Shared logging
│   └── clients/     # Shared ClientProxy configs
└── domain/
    ├── user/
    └── order/
```

---

## 2. Multi-Region Deployment

### 2.1. Architecture

```
                    ┌─────────────────────┐
                    │   Global Load       │
                    │   Balancer (DNS)    │
                    └─────────┬───────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │ Region: US    │ │ Region: EU    │ │ Region: APAC  │
    ├───────────────┤ ├───────────────┤ ├───────────────┤
    │ API Gateway   │ │ API Gateway   │ │ API Gateway   │
    │ Microservices │ │ Microservices │ │ Microservices │
    └───────────────┘ └───────────────┘ └───────────────┘
```

### 2.2. K8s Deployment

```yaml
# k8s/api-gateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api-gateway
          image: myregistry/api-gateway:latest
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
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
```

---

## 3. API Versioning Strategy

### 3.1. URL Path Versioning

```typescript
// api-gateway/src/app.module.ts
import { RouterModule } from '@nestjs/core';

@Module({
  imports: [
    RouterModule.register([
      { path: 'v1', module: V1Module },
      { path: 'v2', module: V2Module },
    ]),
  ],
})
export class AppModule {}
```

### 3.2. Version-Specific Controllers

```typescript
// v1/users.controller.ts
@Controller('users')
export class UsersV1Controller {
  @Get(':id')
  async getUser(@Param('id') id: string) {
    return { id, name: user.name }; // V1 format
  }
}

// v2/users.controller.ts
@Controller('users')
export class UsersV2Controller {
  @Get(':id')
  async getUser(@Param('id') id: string) {
    return { 
      id, 
      profile: { displayName: user.name, avatar: user.avatar },
      metadata: { createdAt: user.createdAt }
    }; // V2 format - enhanced
  }
}
```

---

## 4. Migration Strategies

### 4.1. Strangler Fig Pattern

Migrate từ Monolith sang Microservices từng phần:

```typescript
// api-gateway/src/routing/strangler.service.ts
@Injectable()
export class StranglerRouterService {
  private migrationConfig = {
    '/api/users/*': true,      // Migrated to microservice
    '/api/orders/*': true,     // Migrated
    '/api/products/*': false,  // Still in monolith
  };

  async route(path: string, method: string, body: any) {
    const useMicroservice = this.shouldRoute(path);

    if (useMicroservice) {
      return this.microserviceClient.send(...);
    } else {
      return this.httpService.request({ url: `${MONOLITH_URL}${path}`, method, data: body });
    }
  }
}
```

### 4.2. Blue-Green Deployment

```yaml
# k8s/blue-green.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
spec:
  selector:
    app: api-gateway
    version: blue  # Switch to "green" for new version
  ports:
    - port: 80
      targetPort: 3000
```

---

## 5. Production Observability

### 5.1. Prometheus Metrics

```typescript
// api-gateway/src/metrics/metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram, Registry } from 'prom-client';

@Injectable()
export class MetricsService {
  private registry = new Registry();

  private httpRequestDuration = new Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests',
    labelNames: ['method', 'route', 'status_code'],
    buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
    registers: [this.registry],
  });

  recordRequest(method: string, route: string, statusCode: number, duration: number) {
    this.httpRequestDuration.observe({ method, route, status_code: statusCode }, duration);
  }

  async getMetrics(): Promise<string> {
    return this.registry.metrics();
  }
}
```

### 5.2. Alerting Rules

```yaml
# prometheus/alerts.yaml
groups:
  - name: api-gateway-alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on API Gateway (> 5%)"

      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 2 seconds"
```

---

## Tổng Kết: Decision Matrix

| Scenario | Recommended Solution |
|----------|---------------------|
| Pure NestJS stack | NestJS Native Gateway |
| Multi-language microservices | Kong + NestJS Gateway behind |
| Kubernetes-first | Traefik + NestJS services |
| Serverless/AWS | AWS API Gateway + Lambda/ECS |
| Maximum performance | NGINX + NestJS hybrid |

---

> **Tham khảo:** Xem `RESEARCH_SUMMARY.md` để biết nguồn thông tin và quyết định architecture.
