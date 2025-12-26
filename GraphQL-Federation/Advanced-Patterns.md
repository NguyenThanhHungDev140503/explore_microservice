# Advanced Patterns - GraphQL Federation

> Patterns nâng cao cho Senior developers: Performance optimization, Testing strategies, và Advanced configurations.

## Mục Lục

1. [Custom Configuration](#1-custom-configuration)
2. [Query Key Factory](#2-query-key-factory)
3. [Performance Optimization](#3-performance-optimization)
4. [Testing Strategies](#4-testing-strategies)
5. [Monitoring & Debugging](#5-monitoring--debugging)

---

## 1. Custom Configuration

### 1.1. Custom Apollo Gateway

```typescript
// gateway/src/gateway.config.ts
import { 
  ApolloGateway, 
  IntrospectAndCompose,
  RemoteGraphQLDataSource,
  GraphQLDataSourceProcessOptions
} from '@apollo/gateway';
import { GraphQLError } from 'graphql';

class AuthenticatedDataSource extends RemoteGraphQLDataSource {
  willSendRequest({ request, context }: GraphQLDataSourceProcessOptions) {
    // Forward authentication headers
    if (context.req?.headers.authorization) {
      request.http.headers.set(
        'authorization',
        context.req.headers.authorization
      );
    }

    // Correlation ID for distributed tracing
    const correlationId = context.req?.headers['x-correlation-id'] 
      || crypto.randomUUID();
    request.http.headers.set('x-correlation-id', correlationId);

    // Propagate user info
    if (context.user) {
      request.http.headers.set('x-user-id', context.user.id);
      request.http.headers.set('x-user-roles', JSON.stringify(context.user.roles));
    }
  }

  didReceiveResponse({ response, context }) {
    // Log response metadata
    const serviceName = response.http?.headers.get('x-service-name');
    const responseTime = response.http?.headers.get('x-response-time');
    
    context.logger?.debug(`Response from ${serviceName}: ${responseTime}ms`);
    
    return response;
  }

  didEncounterError(error: Error, fetcherContext: GraphQLDataSourceProcessOptions) {
    // Custom error handling
    console.error(`Gateway error from ${fetcherContext.request.http.url}:`, error);
    
    throw new GraphQLError('Service temporarily unavailable', {
      extensions: {
        code: 'SERVICE_UNAVAILABLE',
        serviceName: this.name,
      },
    });
  }
}

export const createGateway = () => new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: process.env.USERS_SERVICE_URL },
      { name: 'products', url: process.env.PRODUCTS_SERVICE_URL },
      { name: 'orders', url: process.env.ORDERS_SERVICE_URL },
    ],
    pollIntervalInMs: 10000,
    subgraphHealthCheck: true,
  }),
  buildService({ url }) {
    return new AuthenticatedDataSource({ url });
  },
});
```

### 1.2. Global Error Handler

```typescript
// gateway/src/plugins/error-handler.plugin.ts
import { ApolloServerPlugin } from '@apollo/server';

export const ErrorHandlerPlugin: ApolloServerPlugin = {
  async requestDidStart({ contextValue }) {
    return {
      async didEncounterErrors({ errors, contextValue }) {
        for (const error of errors) {
          // Log all errors
          contextValue.logger.error({
            message: error.message,
            path: error.path,
            extensions: error.extensions,
            correlationId: contextValue.correlationId,
          });

          // Alert on critical errors
          if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
            await contextValue.alertService.sendAlert({
              severity: 'critical',
              error,
            });
          }
        }
      },
    };
  },
};
```

### 1.3. Rate Limiting per Subgraph

```typescript
// gateway/src/plugins/rate-limit.plugin.ts
import { ApolloServerPlugin } from '@apollo/server';
import { RateLimiter } from 'limiter';

const limiters = new Map<string, RateLimiter>();

function getLimiter(subgraph: string): RateLimiter {
  if (!limiters.has(subgraph)) {
    limiters.set(subgraph, new RateLimiter({
      tokensPerInterval: 1000,
      interval: 'minute',
    }));
  }
  return limiters.get(subgraph)!;
}

export const RateLimitPlugin: ApolloServerPlugin = {
  async requestDidStart() {
    return {
      async executionDidStart() {
        return {
          async willResolveField({ info }) {
            const subgraph = info.parentType.extensions?.subgraph;
            if (subgraph && !getLimiter(subgraph).tryRemoveTokens(1)) {
              throw new GraphQLError('Rate limit exceeded', {
                extensions: {
                  code: 'RATE_LIMIT_EXCEEDED',
                  subgraph,
                },
              });
            }
          },
        };
      },
    };
  },
};
```

---

## 2. Query Key Factory

### 2.1. Centralized Entity Reference Management

```typescript
// shared/src/federation/entity-ref.factory.ts
export class EntityRefFactory {
  static user(id: string): { __typename: 'User'; id: string } {
    return { __typename: 'User', id };
  }

  static product(id: string): { __typename: 'Product'; id: string } {
    return { __typename: 'Product', id };
  }

  static order(id: string): { __typename: 'Order'; id: string } {
    return { __typename: 'Order', id };
  }

  static productBySku(sku: string): { __typename: 'Product'; sku: string } {
    return { __typename: 'Product', sku };
  }
}
```

### 2.2. Type-Safe Reference Resolution

```typescript
// order-subgraph/src/order.resolver.ts
import { EntityRefFactory } from '@shared/federation';

@Resolver(() => Order)
export class OrderResolver {
  @ResolveField(() => User)
  user(@Parent() order: Order) {
    return EntityRefFactory.user(order.userId);
  }

  @ResolveField(() => [Product])
  products(@Parent() order: Order) {
    return order.productIds.map(id => EntityRefFactory.product(id));
  }
}
```

---

## 3. Performance Optimization

### 3.1. DataLoader Pattern với Federation

```typescript
// user-subgraph/src/user/user.loader.ts
import DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { UserService } from './user.service';
import { User } from './user.model';

@Injectable({ scope: Scope.REQUEST })
export class UserLoader {
  private loader: DataLoader<string, User>;

  constructor(private readonly userService: UserService) {
    this.loader = new DataLoader<string, User>(
      async (ids) => {
        const users = await this.userService.findByIds([...ids]);
        const userMap = new Map(users.map(u => [u.id, u]));
        return ids.map(id => userMap.get(id) || null);
      },
      {
        cache: true,
        maxBatchSize: 100,
      }
    );
  }

  async load(id: string): Promise<User | null> {
    return this.loader.load(id);
  }

  async loadMany(ids: string[]): Promise<(User | null)[]> {
    return this.loader.loadMany(ids);
  }
}

// user.resolver.ts
@Resolver(() => User)
export class UserResolver {
  constructor(private readonly userLoader: UserLoader) {}

  @ResolveReference()
  async resolveReference(ref: { id: string }): Promise<User> {
    return this.userLoader.load(ref.id);
  }
}
```

### 3.2. Query Batching

```typescript
// Gateway supports automatic query batching
// Client-side configuration
const link = new BatchHttpLink({
  uri: 'http://gateway:4000/graphql',
  batchMax: 10,
  batchInterval: 20,
});
```

### 3.3. Response Caching

```typescript
// gateway/src/plugins/cache.plugin.ts
import { ApolloServerPlugin } from '@apollo/server';
import { createHash } from 'crypto';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

export const CachePlugin: ApolloServerPlugin = {
  async requestDidStart({ request }) {
    // Skip mutations
    if (request.operationName?.startsWith('mutation')) {
      return {};
    }

    const cacheKey = createHash('sha256')
      .update(JSON.stringify({
        query: request.query,
        variables: request.variables,
      }))
      .digest('hex');

    return {
      async responseForOperation() {
        const cached = await redis.get(cacheKey);
        if (cached) {
          return JSON.parse(cached);
        }
        return null;
      },

      async willSendResponse({ response }) {
        if (!response.errors) {
          await redis.setex(cacheKey, 60, JSON.stringify(response));
        }
      },
    };
  },
};
```

### 3.4. Selective Field Loading

```typescript
// product-subgraph/src/product.resolver.ts
import { Info } from '@nestjs/graphql';
import { GraphQLResolveInfo } from 'graphql';
import { parseResolveInfo } from 'graphql-parse-resolve-info';

@Resolver(() => Product)
export class ProductResolver {
  @Query(() => Product)
  async product(
    @Args('id') id: string,
    @Info() info: GraphQLResolveInfo,
  ): Promise<Product> {
    const parsedInfo = parseResolveInfo(info);
    const requestedFields = Object.keys(parsedInfo.fieldsByTypeName.Product || {});
    
    // Only fetch requested fields from database
    return this.productService.findById(id, {
      select: requestedFields,
    });
  }
}
```

---

## 4. Testing Strategies

### 4.1. Unit Testing Resolvers

```typescript
// user-subgraph/src/user/user.resolver.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserResolver } from './user.resolver';
import { UserService } from './user.service';

describe('UserResolver', () => {
  let resolver: UserResolver;
  let userService: jest.Mocked<UserService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserResolver,
        {
          provide: UserService,
          useValue: {
            findById: jest.fn(),
            findAll: jest.fn(),
          },
        },
      ],
    }).compile();

    resolver = module.get<UserResolver>(UserResolver);
    userService = module.get(UserService);
  });

  describe('resolveReference', () => {
    it('should resolve user by id', async () => {
      const mockUser = { id: '1', username: 'test', email: 'test@test.com' };
      userService.findById.mockResolvedValue(mockUser);

      const result = await resolver.resolveReference({ 
        __typename: 'User', 
        id: '1' 
      });

      expect(result).toEqual(mockUser);
      expect(userService.findById).toHaveBeenCalledWith('1');
    });

    it('should return null for non-existent user', async () => {
      userService.findById.mockResolvedValue(null);

      const result = await resolver.resolveReference({ 
        __typename: 'User', 
        id: 'non-existent' 
      });

      expect(result).toBeNull();
    });
  });
});
```

### 4.2. Integration Testing với MSW

```typescript
// gateway/test/federation.integration.spec.ts
import { setupServer } from 'msw/node';
import { graphql } from 'msw';
import request from 'supertest';
import { app } from '../src/main';

const handlers = [
  graphql.query('GetUsers', (req, res, ctx) => {
    return res(
      ctx.data({
        users: [
          { id: '1', username: 'test' },
        ],
      })
    );
  }),
];

const server = setupServer(...handlers);

describe('Federation Integration', () => {
  beforeAll(() => server.listen());
  afterEach(() => server.resetHandlers());
  afterAll(() => server.close());

  it('should query users from gateway', async () => {
    const query = `
      query {
        users {
          id
          username
        }
      }
    `;

    const response = await request(app)
      .post('/graphql')
      .send({ query })
      .expect(200);

    expect(response.body.data.users).toHaveLength(1);
  });
});
```

### 4.3. Schema Validation Testing

```typescript
// test/schema-validation.spec.ts
import { buildSubgraphSchema } from '@apollo/subgraph';
import { composeServices } from '@apollo/composition';
import { readFileSync } from 'fs';

describe('Schema Composition', () => {
  it('should compose all subgraphs without errors', () => {
    const services = [
      {
        name: 'users',
        typeDefs: readFileSync('./user-subgraph/schema.graphql', 'utf-8'),
      },
      {
        name: 'products',
        typeDefs: readFileSync('./product-subgraph/schema.graphql', 'utf-8'),
      },
    ];

    const result = composeServices(services);

    expect(result.errors).toBeUndefined();
    expect(result.supergraphSdl).toBeDefined();
  });

  it('should detect breaking changes', () => {
    // Compare current schema with production schema
    // Implement schema diff logic
  });
});
```

---

## 5. Monitoring & Debugging

### 5.1. Distributed Tracing

```typescript
// subgraph/src/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { GraphQLInstrumentation } from '@opentelemetry/instrumentation-graphql';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_COLLECTOR_URL,
  }),
  instrumentations: [
    new GraphQLInstrumentation({
      depth: 3,
      allowValues: true,
    }),
  ],
});

sdk.start();
```

### 5.2. Query Performance Logging

```typescript
// gateway/src/plugins/performance.plugin.ts
import { ApolloServerPlugin } from '@apollo/server';

export const PerformancePlugin: ApolloServerPlugin = {
  async requestDidStart({ request }) {
    const start = Date.now();
    
    return {
      async willSendResponse({ response }) {
        const duration = Date.now() - start;
        
        // Log slow queries
        if (duration > 1000) {
          console.warn({
            type: 'SLOW_QUERY',
            query: request.query?.substring(0, 200),
            duration,
            variables: request.variables,
          });
        }

        // Add timing header
        response.http.headers.set('X-Response-Time', `${duration}ms`);
      },
    };
  },
};
```

### 5.3. Health Check Aggregation

```typescript
// gateway/src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, HttpHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('users-subgraph', 'http://users:3001/health'),
      () => this.http.pingCheck('products-subgraph', 'http://products:3002/health'),
      () => this.http.pingCheck('orders-subgraph', 'http://orders:3003/health'),
    ]);
  }
}
```

---

## Tài Liệu Tham Khảo

- [Apollo Federation Performance](https://www.apollographql.com/docs/federation/performance)
- [DataLoader Pattern](https://github.com/graphql/dataloader)
- [OpenTelemetry GraphQL](https://opentelemetry.io/docs/instrumentation/js/)
