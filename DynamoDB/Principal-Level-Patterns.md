# Principal-Level Patterns - DynamoDB với NestJS

Tài liệu này dành cho **Principal/Staff engineers**, tập trung vào enterprise patterns và system design.

---

## 1. Enterprise Architecture

### 1.1. Centralized DynamoDB Client

```typescript
// dynamodb.module.ts
@Global()
@Module({
  providers: [
    {
      provide: "DYNAMODB_CLIENT",
      useFactory: (config: ConfigService, logger: Logger) => {
        const client = new DynamoDBClient({
          region: config.get("AWS_REGION"),
          maxAttempts: 3,
          logger: {
            trace: () => {},
            debug: (msg) => logger.debug(msg),
            info: (msg) => logger.log(msg),
            warn: (msg) => logger.warn(msg),
            error: (msg) => logger.error(msg),
          },
        });
        
        // Middleware cho request/response logging
        client.middlewareStack.add(
          (next) => async (args) => {
            const start = Date.now();
            const result = await next(args);
            logger.log(`DynamoDB ${args.input.constructor.name} took ${Date.now() - start}ms`);
            return result;
          },
          { step: "deserialize", name: "loggingMiddleware" }
        );
        
        return client;
      },
      inject: [ConfigService, Logger],
    },
  ],
  exports: ["DYNAMODB_CLIENT"],
})
export class DynamoDBModule {}
```

### 1.2. Multi-Table Strategy

```typescript
// Khi nào dùng Single vs Multi-Table
// Single Table: Khi entities có chung access patterns
// Multi-Table: Khi cần isolation, different scaling, security

interface TableConfig {
  name: string;
  purpose: "transactional" | "analytics" | "cache";
  capacityMode: "on-demand" | "provisioned";
}

const tables: TableConfig[] = [
  { name: "Orders", purpose: "transactional", capacityMode: "on-demand" },
  { name: "Analytics", purpose: "analytics", capacityMode: "provisioned" },
  { name: "Sessions", purpose: "cache", capacityMode: "on-demand" },
];
```

---

## 2. Global Tables & Multi-Region

### 2.1. Global Tables Architecture

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   ap-southeast-1 │◄──►│   us-east-1      │◄──►│   eu-west-1      │
│   (Singapore)    │    │   (Virginia)     │    │   (Ireland)      │
│                  │    │                  │    │                  │
│   ┌──────────┐   │    │   ┌──────────┐   │    │   ┌──────────┐   │
│   │  Table   │   │    │   │  Table   │   │    │   │  Table   │   │
│   │ (Replica)│   │    │   │ (Replica)│   │    │   │ (Replica)│   │
│   └──────────┘   │    │   └──────────┘   │    │   └──────────┘   │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

### 2.2. Conflict Resolution

```typescript
// DynamoDB Global Tables dùng last-writer-wins
// Để handle conflicts, thêm timestamp và region

const item = {
  pk: "USER#123",
  sk: "PROFILE",
  name: "John",
  updatedAt: Date.now(),
  updatedRegion: process.env.AWS_REGION,
};

// Read với strong consistency khi cần
const command = new GetCommand({
  TableName: "GlobalTable",
  Key: { pk: "USER#123", sk: "PROFILE" },
  ConsistentRead: true,  // Strong consistency trong region
});
```

---

## 3. Event Sourcing với DynamoDB Streams

### 3.1. Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  DynamoDB   │───►│  Streams    │───►│   Lambda    │
│   Table     │    │             │    │  (Handler)  │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                              │
        ┌─────────────────────────────────────┼─────────────────┐
        │                                     │                 │
        ▼                                     ▼                 ▼
┌───────────────┐                    ┌───────────────┐  ┌───────────────┐
│ Elasticsearch │                    │     SNS       │  │   EventBridge │
│   (Search)    │                    │  (Notify)     │  │   (Events)    │
└───────────────┘                    └───────────────┘  └───────────────┘
```

### 3.2. Stream Handler

```typescript
// Lambda handler cho DynamoDB Streams
export async function handler(event: DynamoDBStreamEvent) {
  for (const record of event.Records) {
    const newImage = record.dynamodb?.NewImage 
      ? unmarshall(record.dynamodb.NewImage as any) 
      : null;
    const oldImage = record.dynamodb?.OldImage 
      ? unmarshall(record.dynamodb.OldImage as any) 
      : null;

    switch (record.eventName) {
      case "INSERT":
        await handleInsert(newImage);
        break;
      case "MODIFY":
        await handleUpdate(newImage, oldImage);
        break;
      case "REMOVE":
        await handleDelete(oldImage);
        break;
    }
  }
}

async function handleInsert(item: any) {
  // Sync to search index, send notifications, etc.
  await elasticsearch.index({ index: "items", body: item });
}
```

---

## 4. Migration Strategies

### 4.1. Zero-Downtime Migration

```
Phase 1: Dual Write
┌─────────┐    ┌─────────────┐
│   App   │───►│  Old Table  │
│         │───►│  New Table  │
└─────────┘    └─────────────┘

Phase 2: Migrate Historical Data
┌─────────────┐    ┌─────────────┐
│  Old Table  │───►│  New Table  │
│             │    │ (via script)│
└─────────────┘    └─────────────┘

Phase 3: Switch Read
┌─────────┐    ┌─────────────┐
│   App   │◄───│  New Table  │
│         │───►│  Old Table  │
└─────────┘    └─────────────┘

Phase 4: Complete Cutover
┌─────────┐    ┌─────────────┐
│   App   │◄──►│  New Table  │
└─────────┘    └─────────────┘
```

### 4.2. Migration Script

```typescript
async function migrateData(sourceTable: string, targetTable: string) {
  let lastKey: any;
  let migratedCount = 0;

  do {
    const scanResult = await docClient.send(new ScanCommand({
      TableName: sourceTable,
      ExclusiveStartKey: lastKey,
      Limit: 100,
    }));

    if (scanResult.Items?.length) {
      const batches = chunk(scanResult.Items, 25);
      for (const batch of batches) {
        await docClient.send(new BatchWriteCommand({
          RequestItems: {
            [targetTable]: batch.map(item => ({
              PutRequest: { Item: transformItem(item) },
            })),
          },
        }));
        migratedCount += batch.length;
        console.log(`Migrated ${migratedCount} items`);
      }
    }

    lastKey = scanResult.LastEvaluatedKey;
  } while (lastKey);
}

function transformItem(item: any): any {
  // Transform old schema to new schema
  return { ...item, version: 2 };
}
```

---

## 5. Monitoring & Observability

### 5.1. CloudWatch Metrics

```typescript
// Key metrics to monitor
const criticalMetrics = [
  "ConsumedReadCapacityUnits",
  "ConsumedWriteCapacityUnits",
  "ThrottledRequests",
  "SystemErrors",
  "SuccessfulRequestLatency",
];

// CloudWatch Alarm cho throttling
const alarmConfig = {
  AlarmName: "DynamoDB-Throttling",
  MetricName: "ThrottledRequests",
  Namespace: "AWS/DynamoDB",
  Threshold: 10,
  EvaluationPeriods: 1,
  Period: 60,
  ComparisonOperator: "GreaterThanThreshold",
};
```

### 5.2. Custom Metrics Module

```typescript
@Injectable()
export class DynamoDBMetricsService {
  constructor(
    private readonly cloudwatch: CloudWatchClient,
    @Inject("DYNAMODB_DOC_CLIENT") private readonly docClient: DynamoDBDocumentClient,
  ) {}

  async trackOperation(operation: string, tableName: string, fn: () => Promise<any>) {
    const start = Date.now();
    try {
      const result = await fn();
      await this.publishMetric(operation, tableName, Date.now() - start, true);
      return result;
    } catch (error) {
      await this.publishMetric(operation, tableName, Date.now() - start, false);
      throw error;
    }
  }

  private async publishMetric(operation: string, table: string, duration: number, success: boolean) {
    await this.cloudwatch.send(new PutMetricDataCommand({
      Namespace: "Custom/DynamoDB",
      MetricData: [
        { MetricName: "OperationDuration", Value: duration, Unit: "Milliseconds",
          Dimensions: [{ Name: "Operation", Value: operation }, { Name: "Table", Value: table }] },
        { MetricName: success ? "SuccessCount" : "ErrorCount", Value: 1, Unit: "Count",
          Dimensions: [{ Name: "Operation", Value: operation }] },
      ],
    }));
  }
}
```

---

## 6. Best Practices Summary

| Category | Best Practice |
|----------|---------------|
| **Design** | Design cho access patterns trước, schema sau |
| **Keys** | Distribute partition key evenly |
| **Capacity** | Dùng On-Demand cho unpredictable workloads |
| **Indexes** | GSI > LSI cho flexibility |
| **Queries** | Query > Scan luôn luôn |
| **Caching** | DAX cho read-heavy workloads |
| **Monitoring** | Alert on ThrottledRequests |
| **Cost** | Review unused GSIs regularly |

---

## Tham Khảo

- [AWS Well-Architected - DynamoDB](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/dynamodb.html)
- [AWS DynamoDB Global Tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)
- [DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
