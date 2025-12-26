# Advanced Patterns: NestJS API Gateway

> **Dành cho:** Senior Developer  
> **Nội dung:** Performance optimization, distributed tracing, testing strategies, caching

---

## Mục Lục

1. [Performance Optimization](#1-performance-optimization)
2. [Distributed Tracing](#2-distributed-tracing)
3. [Caching Strategies](#3-caching-strategies)
4. [Testing Strategies](#4-testing-strategies)
5. [Security Hardening](#5-security-hardening)

---

## 1. Performance Optimization

### 1.1. Connection Pooling

```typescript
// api-gateway/src/config/clients.config.ts
import { ClientsModuleOptions, Transport } from '@nestjs/microservices';

export const clientsConfig: ClientsModuleOptions = [
  {
    name: 'USER_SERVICE',
    transport: Transport.TCP,
    options: {
      host: process.env.USER_SERVICE_HOST,
      port: parseInt(process.env.USER_SERVICE_PORT),
      retryAttempts: 5,
      retryDelay: 1000,
    },
  },
];
```

### 1.2. Request Batching với DataLoader

```typescript
// api-gateway/src/common/user-loader.service.ts
import { Injectable } from '@nestjs/common';
import DataLoader from 'dataloader';

@Injectable()
export class UserLoaderService {
  private loader = new DataLoader<string, User>(
    async (ids: readonly string[]) => {
      // Batch multiple user requests into one call
      const users = await this.userService.findByIds([...ids]);
      return ids.map((id) => users.find((u) => u.id === id) || null);
    },
    { 
      maxBatchSize: 100, 
      batchScheduleFn: (cb) => setTimeout(cb, 10) 
    },
  );

  async getUser(id: string): Promise<User> {
    return this.loader.load(id);
  }
}
```

### 1.3. Response Compression

```typescript
// api-gateway/src/main.ts
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.use(compression({
    level: 6,
    filter: (req, res) => {
      if (req.headers['x-no-compression']) return false;
      return compression.filter(req, res);
    },
  }));

  await app.listen(3000);
}
```

---

## 2. Distributed Tracing

### 2.1. OpenTelemetry Setup

```typescript
// api-gateway/src/tracing/tracing.service.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';

export function initTracing() {
  const sdk = new NodeSDK({
    traceExporter: new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_ENDPOINT || 'http://jaeger:4318/v1/traces',
    }),
    instrumentations: [new HttpInstrumentation()],
    serviceName: 'api-gateway',
  });

  sdk.start();
  return sdk;
}
```

### 2.2. Correlation ID Middleware

```typescript
// api-gateway/src/common/correlation-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = req.headers['x-correlation-id'] as string || uuidv4();
    
    req['correlationId'] = correlationId;
    res.setHeader('x-correlation-id', correlationId);
    
    next();
  }
}
```

---

## 3. Caching Strategies

### 3.1. Redis Cache

```typescript
// api-gateway/src/cache/cache.module.ts
import { CacheModule } from '@nestjs/cache-manager';
import { redisStore } from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.registerAsync({
      useFactory: async () => ({
        store: await redisStore({
          socket: {
            host: process.env.REDIS_HOST,
            port: parseInt(process.env.REDIS_PORT),
          },
          ttl: 300, // 5 minutes default
        }),
      }),
    }),
  ],
})
export class CacheConfigModule {}
```

### 3.2. Controller với Cache

```typescript
// api-gateway/src/users/users.controller.ts
import { CacheInterceptor, CacheTTL, CacheKey } from '@nestjs/cache-manager';

@Controller('users')
@UseInterceptors(CacheInterceptor)
export class UsersController {
  @Get(':id')
  @CacheTTL(60) // Cache 60 seconds
  @CacheKey('user')
  async getUser(@Param('id') id: string) {
    return this.userClient.send({ cmd: 'get_user' }, { id });
  }
}
```

---

## 4. Testing Strategies

### 4.1. Unit Testing

```typescript
// api-gateway/src/users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { of } from 'rxjs';

describe('UsersController', () => {
  let controller: UsersController;
  let mockUserClient: { send: jest.Mock };

  beforeEach(async () => {
    mockUserClient = { send: jest.fn() };

    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        { provide: 'USER_SERVICE', useValue: mockUserClient },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
  });

  describe('getUser', () => {
    it('should return user from microservice', async () => {
      const mockUser = { id: '1', name: 'Test' };
      mockUserClient.send.mockReturnValue(of(mockUser));

      const result = await controller.getUser('1');

      expect(mockUserClient.send).toHaveBeenCalledWith(
        { cmd: 'get_user' },
        { id: '1' },
      );
      expect(result).toEqual(mockUser);
    });
  });
});
```

### 4.2. E2E Testing

```typescript
// api-gateway/test/gateway.e2e-spec.ts
import * as request from 'supertest';

describe('API Gateway (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/users/:id (GET)', () => {
    return request(app.getHttpServer())
      .get('/users/1')
      .set('Authorization', 'Bearer valid-token')
      .expect(200)
      .expect((res) => {
        expect(res.body).toHaveProperty('id');
      });
  });
});
```

---

## 5. Security Hardening

### 5.1. Helmet Security Headers

```typescript
// api-gateway/src/main.ts
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true },
}));
```

### 5.2. CORS Configuration

```typescript
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Correlation-ID'],
  credentials: true,
  maxAge: 86400,
});
```

---

> **Tiếp theo:** Đọc `Principal-Level-Patterns.md` cho enterprise architecture patterns.
