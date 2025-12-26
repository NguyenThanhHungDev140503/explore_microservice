# DynamoDB - Hướng Dẫn Toàn Diện

## Mục Lục

1. [Giới Thiệu Tổng Quan](#1-giới-thiệu-tổng-quan)
2. [Junior Level – Cơ Bản](#2-junior-level--cơ-bản)
3. [Middle Level – Trung Cấp](#3-middle-level--trung-cấp)
4. [Senior Level – Nâng Cao](#4-senior-level--nâng-cao)
5. [Tích Hợp với NestJS](#5-tích-hợp-với-nestjs)
6. [Tài Liệu Tham Khảo](#6-tài-liệu-tham-khảo)

---

## 1. Giới Thiệu Tổng Quan

### 1.1. DynamoDB là gì?

**Amazon DynamoDB** là một **fully managed NoSQL database service** của AWS, cung cấp tốc độ nhanh và khả năng mở rộng linh hoạt cho mọi quy mô ứng dụng.

#### Đặc điểm chính:
- **NoSQL Database**: Key-value và document database
- **Serverless**: Không cần quản lý servers, infrastructure
- **Fully Managed**: AWS quản lý toàn bộ operations
- **Multi-Region**: Global Tables với replication tự động
- **High Performance**: Single-digit millisecond latency

### 1.2. Tại sao sử dụng DynamoDB?

| Lý do | Mô tả |
|-------|-------|
| **Performance** | Latency dưới 10ms ở mọi scale |
| **Scalability** | Auto-scaling không giới hạn |
| **Availability** | 99.999% SLA với Global Tables |
| **Cost** | Pay-per-request hoặc provisioned capacity |
| **AWS Integration** | Tích hợp sâu với Lambda, API Gateway, etc. |

### 1.3. Các Khái Niệm Cốt Lõi

```
┌─────────────────────────────────────────────────────────────────┐
│                          TABLE                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                         ITEM                             │    │
│  │  ┌──────────────────┬─────────────────────────────────┐ │    │
│  │  │ PRIMARY KEY      │ ATTRIBUTES                      │ │    │
│  │  ├──────────────────┼─────────────────────────────────┤ │    │
│  │  │ pk: "USER#123"   │ name: "John"                    │ │    │
│  │  │ sk: "PROFILE"    │ email: "john@example.com"       │ │    │
│  │  │                  │ createdAt: "2024-01-01"         │ │    │
│  │  └──────────────────┴─────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ pk: "USER#123" | sk: "ORDER#001" | total: 150.00       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

#### Giải thích:
- **Table**: Container cho data, tương tự như table trong SQL
- **Item**: Một record, tương tự như row trong SQL (max 400KB)
- **Attribute**: Field của item, tương tự như column
- **Primary Key**: Định danh unique cho item
  - **Partition Key (PK)**: Required, quyết định partition lưu trữ
  - **Sort Key (SK)**: Optional, cho phép range queries

---

## 2. Junior Level – Cơ Bản

### 2.1. Cài Đặt và Cấu Hình

#### Cài đặt AWS SDK v3

```bash
# Cài đặt các packages cần thiết
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

#### Cấu hình AWS Credentials

```bash
# ~/.aws/credentials
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY

# ~/.aws/config
[default]
region = ap-southeast-1
```

#### Tạo DynamoDB Client

```typescript
// dynamodb.client.ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

// Low-level client
const client = new DynamoDBClient({ 
  region: "ap-southeast-1" 
});

// Document client (high-level, marshalling tự động)
const docClient = DynamoDBDocumentClient.from(client);

export { client, docClient };
```

### 2.2. CRUD Operations Cơ Bản

#### PutItem - Tạo mới item

```typescript
import { PutCommand } from "@aws-sdk/lib-dynamodb";
import { docClient } from "./dynamodb.client";

async function createUser(id: string, name: string, email: string) {
  const command = new PutCommand({
    TableName: "Users",
    Item: {
      id,              // Partition key
      name,
      email,
      createdAt: new Date().toISOString(),
    },
  });

  await docClient.send(command);
  console.log("User created successfully");
}

// Sử dụng
await createUser("user-123", "John Doe", "john@example.com");
```

#### GetItem - Lấy một item

```typescript
import { GetCommand } from "@aws-sdk/lib-dynamodb";
import { docClient } from "./dynamodb.client";

async function getUser(id: string) {
  const command = new GetCommand({
    TableName: "Users",
    Key: {
      id,  // Primary key
    },
  });

  const response = await docClient.send(command);
  return response.Item;
}

// Sử dụng
const user = await getUser("user-123");
console.log(user);
// { id: "user-123", name: "John Doe", email: "john@example.com", createdAt: "..." }
```

#### UpdateItem - Cập nhật item

```typescript
import { UpdateCommand } from "@aws-sdk/lib-dynamodb";
import { docClient } from "./dynamodb.client";

async function updateUserEmail(id: string, newEmail: string) {
  const command = new UpdateCommand({
    TableName: "Users",
    Key: { id },
    UpdateExpression: "SET email = :email, updatedAt = :updatedAt",
    ExpressionAttributeValues: {
      ":email": newEmail,
      ":updatedAt": new Date().toISOString(),
    },
    ReturnValues: "ALL_NEW",  // Trả về item sau khi update
  });

  const response = await docClient.send(command);
  return response.Attributes;
}

// Sử dụng
const updatedUser = await updateUserEmail("user-123", "newemail@example.com");
```

#### DeleteItem - Xóa item

```typescript
import { DeleteCommand } from "@aws-sdk/lib-dynamodb";
import { docClient } from "./dynamodb.client";

async function deleteUser(id: string) {
  const command = new DeleteCommand({
    TableName: "Users",
    Key: { id },
    ReturnValues: "ALL_OLD",  // Trả về item trước khi xóa
  });

  const response = await docClient.send(command);
  return response.Attributes;
}

// Sử dụng
const deletedUser = await deleteUser("user-123");
```

### 2.3. Common Use Cases

#### 1. User Management

```typescript
// Lưu user với profile
const userItem = {
  pk: "USER#user-123",
  sk: "PROFILE",
  name: "John Doe",
  email: "john@example.com",
  role: "admin",
  entityType: "User",
};
```

#### 2. Session Storage

```typescript
// Lưu session với TTL
const sessionItem = {
  pk: "SESSION#abc123",
  sk: "SESSION",
  userId: "user-123",
  token: "jwt-token-here",
  ttl: Math.floor(Date.now() / 1000) + 3600,  // 1 hour TTL
};
```

#### 3. Configuration Storage

```typescript
// Lưu app configuration
const configItem = {
  pk: "CONFIG",
  sk: "APP_SETTINGS",
  featureFlags: {
    darkMode: true,
    betaFeatures: false,
  },
  version: "1.0.0",
};
```

### 2.4. Common Pitfalls (Lỗi Thường Gặp)

#### ❌ Sai: Dùng Scan thay vì Query

```typescript
// ❌ KHÔNG NÊN - Scan quét toàn bộ table, tốn RCU và chậm
import { ScanCommand } from "@aws-sdk/lib-dynamodb";

async function findUserByEmail(email: string) {
  const command = new ScanCommand({
    TableName: "Users",
    FilterExpression: "email = :email",
    ExpressionAttributeValues: { ":email": email },
  });
  const response = await docClient.send(command);
  return response.Items?.[0];
}
```

```typescript
// ✅ NÊN LÀM - Tạo GSI trên email và dùng Query
import { QueryCommand } from "@aws-sdk/lib-dynamodb";

async function findUserByEmail(email: string) {
  const command = new QueryCommand({
    TableName: "Users",
    IndexName: "email-index",  // GSI
    KeyConditionExpression: "email = :email",
    ExpressionAttributeValues: { ":email": email },
  });
  const response = await docClient.send(command);
  return response.Items?.[0];
}
```

#### ❌ Sai: Không handle pagination

```typescript
// ❌ KHÔNG NÊN - Chỉ lấy được tối đa 1MB data
async function getAllUsers() {
  const command = new ScanCommand({ TableName: "Users" });
  const response = await docClient.send(command);
  return response.Items;
}
```

```typescript
// ✅ NÊN LÀM - Handle pagination
async function getAllUsers() {
  const items: any[] = [];
  let lastEvaluatedKey: Record<string, any> | undefined;

  do {
    const command = new ScanCommand({
      TableName: "Users",
      ExclusiveStartKey: lastEvaluatedKey,
    });
    const response = await docClient.send(command);
    
    if (response.Items) {
      items.push(...response.Items);
    }
    lastEvaluatedKey = response.LastEvaluatedKey;
  } while (lastEvaluatedKey);

  return items;
}
```

#### ❌ Sai: Lưu item quá lớn

```typescript
// ❌ KHÔNG NÊN - Item có thể vượt quá 400KB limit
const item = {
  id: "user-123",
  largeData: someLargeObject,  // Có thể > 400KB
};
```

```typescript
// ✅ NÊN LÀM - Lưu large data riêng (S3) và reference
const item = {
  id: "user-123",
  largeDataS3Key: "s3://bucket/user-123/data.json",
};
```

### 2.5. Best Practices Cơ Bản

1. **Luôn dùng Query thay vì Scan** khi có thể
2. **Design schema dựa trên access patterns** trước
3. **Sử dụng Composite Sort Keys** cho hierarchical data
4. **Enable TTL** cho temporary data (sessions, logs)
5. **Sử dụng Condition Expressions** cho optimistic locking
6. **Monitor với CloudWatch** để track performance
7. **Sử dụng DynamoDB Local** cho development
8. **Batch operations** cho bulk reads/writes
9. **Tránh hot partitions** - distribute workload đều
10. **Document access patterns** trước khi design table

---

## 3. Middle Level – Trung Cấp

### 3.1. Query Operations

#### Query với Sort Key

```typescript
import { QueryCommand } from "@aws-sdk/lib-dynamodb";

// Lấy tất cả orders của một user
async function getUserOrders(userId: string) {
  const command = new QueryCommand({
    TableName: "MainTable",
    KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
    ExpressionAttributeValues: {
      ":pk": `USER#${userId}`,
      ":skPrefix": "ORDER#",
    },
  });

  const response = await docClient.send(command);
  return response.Items;
}
```

#### Query với Filter

```typescript
// Lấy orders theo status
async function getUserOrdersByStatus(userId: string, status: string) {
  const command = new QueryCommand({
    TableName: "MainTable",
    KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
    FilterExpression: "#status = :status",
    ExpressionAttributeNames: {
      "#status": "status",  // "status" là reserved word
    },
    ExpressionAttributeValues: {
      ":pk": `USER#${userId}`,
      ":skPrefix": "ORDER#",
      ":status": status,
    },
  });

  return (await docClient.send(command)).Items;
}
```

#### Query với Pagination

```typescript
interface PaginatedResult<T> {
  items: T[];
  nextToken?: string;
}

async function getUserOrdersPaginated(
  userId: string,
  limit: number = 10,
  nextToken?: string
): Promise<PaginatedResult<any>> {
  const command = new QueryCommand({
    TableName: "MainTable",
    KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
    ExpressionAttributeValues: {
      ":pk": `USER#${userId}`,
      ":skPrefix": "ORDER#",
    },
    Limit: limit,
    ExclusiveStartKey: nextToken 
      ? JSON.parse(Buffer.from(nextToken, "base64").toString()) 
      : undefined,
  });

  const response = await docClient.send(command);

  return {
    items: response.Items || [],
    nextToken: response.LastEvaluatedKey
      ? Buffer.from(JSON.stringify(response.LastEvaluatedKey)).toString("base64")
      : undefined,
  };
}
```

### 3.2. Secondary Indexes

#### Global Secondary Index (GSI)

```typescript
// GSI cho phép query trên non-key attributes
// Ví dụ: Tìm orders theo status

// Table design:
// pk: USER#123, sk: ORDER#001, status: "pending", GSI1PK: "STATUS#pending", GSI1SK: "2024-01-01"

async function getOrdersByStatus(status: string) {
  const command = new QueryCommand({
    TableName: "MainTable",
    IndexName: "GSI1",  // GSI name
    KeyConditionExpression: "GSI1PK = :gsi1pk",
    ExpressionAttributeValues: {
      ":gsi1pk": `STATUS#${status}`,
    },
  });

  return (await docClient.send(command)).Items;
}
```

#### Local Secondary Index (LSI)

```typescript
// LSI phải có cùng partition key với base table
// Ví dụ: Query user orders theo date

async function getUserOrdersByDate(userId: string, startDate: string, endDate: string) {
  const command = new QueryCommand({
    TableName: "MainTable",
    IndexName: "LSI1",  // LSI name
    KeyConditionExpression: "pk = :pk AND orderDate BETWEEN :start AND :end",
    ExpressionAttributeValues: {
      ":pk": `USER#${userId}`,
      ":start": startDate,
      ":end": endDate,
    },
  });

  return (await docClient.send(command)).Items;
}
```

#### So sánh GSI vs LSI

| Feature | GSI | LSI |
|---------|-----|-----|
| Partition Key | Khác base table được | Phải giống base table |
| Tạo khi nào | Bất cứ lúc nào | Chỉ khi tạo table |
| Số lượng | Max 20/table | Max 5/table |
| Consistency | Eventually consistent | Eventually hoặc Strong |
| Provisioned Capacity | Riêng biệt | Dùng chung với table |

### 3.3. Batch Operations

#### BatchGetItem

```typescript
import { BatchGetCommand } from "@aws-sdk/lib-dynamodb";

async function getUsersBatch(userIds: string[]) {
  const command = new BatchGetCommand({
    RequestItems: {
      Users: {
        Keys: userIds.map(id => ({ id })),
      },
    },
  });

  const response = await docClient.send(command);
  return response.Responses?.Users || [];
}
```

#### BatchWriteItem

```typescript
import { BatchWriteCommand } from "@aws-sdk/lib-dynamodb";

async function createUsersBatch(users: any[]) {
  // BatchWrite tối đa 25 items
  const batches = [];
  for (let i = 0; i < users.length; i += 25) {
    batches.push(users.slice(i, i + 25));
  }

  for (const batch of batches) {
    const command = new BatchWriteCommand({
      RequestItems: {
        Users: batch.map(user => ({
          PutRequest: { Item: user },
        })),
      },
    });

    const response = await docClient.send(command);
    
    // Handle unprocessed items (throttling)
    if (response.UnprocessedItems && Object.keys(response.UnprocessedItems).length > 0) {
      // Retry với exponential backoff
      console.log("Some items were not processed, retrying...");
    }
  }
}
```

### 3.4. Conditional Writes

#### Optimistic Locking với Version

```typescript
async function updateUserWithVersion(id: string, updates: any, expectedVersion: number) {
  const command = new UpdateCommand({
    TableName: "Users",
    Key: { id },
    UpdateExpression: "SET #name = :name, version = :newVersion",
    ConditionExpression: "version = :expectedVersion",
    ExpressionAttributeNames: {
      "#name": "name",
    },
    ExpressionAttributeValues: {
      ":name": updates.name,
      ":expectedVersion": expectedVersion,
      ":newVersion": expectedVersion + 1,
    },
    ReturnValues: "ALL_NEW",
  });

  try {
    const response = await docClient.send(command);
    return response.Attributes;
  } catch (error: any) {
    if (error.name === "ConditionalCheckFailedException") {
      throw new Error("Item was modified by another process");
    }
    throw error;
  }
}
```

#### Prevent Overwrites

```typescript
async function createUserIfNotExists(user: any) {
  const command = new PutCommand({
    TableName: "Users",
    Item: user,
    ConditionExpression: "attribute_not_exists(id)",  // Chỉ insert nếu chưa tồn tại
  });

  try {
    await docClient.send(command);
    return true;
  } catch (error: any) {
    if (error.name === "ConditionalCheckFailedException") {
      return false;  // User already exists
    }
    throw error;
  }
}
```

### 3.5. Error Handling

```typescript
import { 
  ConditionalCheckFailedException,
  ProvisionedThroughputExceededException,
  ResourceNotFoundException,
} from "@aws-sdk/client-dynamodb";

async function safeGetUser(id: string) {
  try {
    const command = new GetCommand({
      TableName: "Users",
      Key: { id },
    });
    return await docClient.send(command);
  } catch (error) {
    if (error instanceof ResourceNotFoundException) {
      console.error("Table not found");
      throw new Error("Database configuration error");
    }
    
    if (error instanceof ProvisionedThroughputExceededException) {
      console.error("Throughput exceeded, retrying...");
      // Implement exponential backoff
      await sleep(1000);
      return safeGetUser(id);  // Retry
    }
    
    if (error instanceof ConditionalCheckFailedException) {
      console.error("Condition check failed");
      throw new Error("Concurrent modification detected");
    }
    
    throw error;
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## 4. Senior Level – Nâng Cao

### 4.1. Single Table Design

#### Concept

Single Table Design là pattern lưu nhiều entity types trong một table, sử dụng composite keys để phân biệt.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ pk                  │ sk                   │ Attributes                  │
├──────────────────────────────────────────────────────────────────────────┤
│ USER#user-123       │ PROFILE              │ name, email, role           │
│ USER#user-123       │ ORDER#order-001      │ total, status, createdAt    │
│ USER#user-123       │ ORDER#order-002      │ total, status, createdAt    │
│ ORDER#order-001     │ ORDER#order-001      │ userId, items, total        │
│ ORDER#order-001     │ ITEM#item-001        │ productId, quantity, price  │
│ PRODUCT#prod-001    │ PRODUCT#prod-001     │ name, price, category       │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Entity Modeling

```typescript
// Entity types
type EntityType = "User" | "Order" | "OrderItem" | "Product";

interface BaseEntity {
  pk: string;
  sk: string;
  entityType: EntityType;
  createdAt: string;
  updatedAt: string;
}

interface User extends BaseEntity {
  entityType: "User";
  userId: string;
  name: string;
  email: string;
  role: "admin" | "user";
}

interface Order extends BaseEntity {
  entityType: "Order";
  orderId: string;
  userId: string;
  status: "pending" | "processing" | "shipped" | "delivered";
  total: number;
}

// Key factories
const keys = {
  user: {
    pk: (userId: string) => `USER#${userId}`,
    sk: () => "PROFILE",
  },
  order: {
    pk: (userId: string) => `USER#${userId}`,
    sk: (orderId: string) => `ORDER#${orderId}`,
  },
  orderDetail: {
    pk: (orderId: string) => `ORDER#${orderId}`,
    sk: (orderId: string) => `ORDER#${orderId}`,
  },
};
```

### 4.2. Transaction Operations

#### TransactWriteItems

```typescript
import { TransactWriteCommand } from "@aws-sdk/lib-dynamodb";

async function createOrderWithItems(order: any, items: any[]) {
  const transactItems = [
    // Create order
    {
      Put: {
        TableName: "MainTable",
        Item: {
          pk: `USER#${order.userId}`,
          sk: `ORDER#${order.orderId}`,
          ...order,
          entityType: "Order",
        },
        ConditionExpression: "attribute_not_exists(pk)",
      },
    },
    // Create order items
    ...items.map(item => ({
      Put: {
        TableName: "MainTable",
        Item: {
          pk: `ORDER#${order.orderId}`,
          sk: `ITEM#${item.itemId}`,
          ...item,
          entityType: "OrderItem",
        },
      },
    })),
    // Update user order count
    {
      Update: {
        TableName: "MainTable",
        Key: {
          pk: `USER#${order.userId}`,
          sk: "PROFILE",
        },
        UpdateExpression: "ADD orderCount :inc",
        ExpressionAttributeValues: {
          ":inc": 1,
        },
      },
    },
  ];

  const command = new TransactWriteCommand({
    TransactItems: transactItems,
  });

  await docClient.send(command);
}
```

#### TransactGetItems

```typescript
import { TransactGetCommand } from "@aws-sdk/lib-dynamodb";

async function getOrderWithUser(orderId: string, userId: string) {
  const command = new TransactGetCommand({
    TransactItems: [
      {
        Get: {
          TableName: "MainTable",
          Key: {
            pk: `USER#${userId}`,
            sk: "PROFILE",
          },
        },
      },
      {
        Get: {
          TableName: "MainTable",
          Key: {
            pk: `USER#${userId}`,
            sk: `ORDER#${orderId}`,
          },
        },
      },
    ],
  });

  const response = await docClient.send(command);
  const [userResponse, orderResponse] = response.Responses || [];

  return {
    user: userResponse?.Item,
    order: orderResponse?.Item,
  };
}
```

### 4.3. DynamoDB Streams

```typescript
// Lambda handler cho DynamoDB Streams
import { DynamoDBStreamEvent } from "aws-lambda";
import { unmarshall } from "@aws-sdk/util-dynamodb";

export async function handler(event: DynamoDBStreamEvent) {
  for (const record of event.Records) {
    const eventName = record.eventName;  // INSERT, MODIFY, REMOVE
    
    const newImage = record.dynamodb?.NewImage 
      ? unmarshall(record.dynamodb.NewImage as any) 
      : null;
    const oldImage = record.dynamodb?.OldImage 
      ? unmarshall(record.dynamodb.OldImage as any) 
      : null;

    switch (eventName) {
      case "INSERT":
        console.log("New item:", newImage);
        // Handle new item (e.g., send notification)
        break;
      case "MODIFY":
        console.log("Updated item:", newImage, "Old:", oldImage);
        // Handle update (e.g., sync to search index)
        break;
      case "REMOVE":
        console.log("Deleted item:", oldImage);
        // Handle delete (e.g., cleanup related data)
        break;
    }
  }
}
```

### 4.4. Performance Optimization

#### Parallel Scan

```typescript
async function parallelScan(tableName: string, segments: number = 4) {
  const scanPromises = Array.from({ length: segments }, (_, segment) =>
    scanSegment(tableName, segment, segments)
  );

  const results = await Promise.all(scanPromises);
  return results.flat();
}

async function scanSegment(tableName: string, segment: number, totalSegments: number) {
  const items: any[] = [];
  let lastEvaluatedKey: any;

  do {
    const command = new ScanCommand({
      TableName: tableName,
      Segment: segment,
      TotalSegments: totalSegments,
      ExclusiveStartKey: lastEvaluatedKey,
    });

    const response = await docClient.send(command);
    items.push(...(response.Items || []));
    lastEvaluatedKey = response.LastEvaluatedKey;
  } while (lastEvaluatedKey);

  return items;
}
```

#### Projection Expressions

```typescript
// Chỉ lấy các attributes cần thiết
async function getUserSummary(userId: string) {
  const command = new GetCommand({
    TableName: "Users",
    Key: { id: userId },
    ProjectionExpression: "id, #name, email",  // Chỉ lấy 3 fields
    ExpressionAttributeNames: {
      "#name": "name",  // "name" là reserved word
    },
  });

  return (await docClient.send(command)).Item;
}
```

---

## 5. Tích Hợp với NestJS

### 5.1. Sử dụng AWS SDK v3

#### DynamoDB Module

```typescript
// dynamodb.module.ts
import { Module, Global } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

export const DYNAMODB_CLIENT = "DYNAMODB_CLIENT";
export const DYNAMODB_DOC_CLIENT = "DYNAMODB_DOC_CLIENT";

@Global()
@Module({
  providers: [
    {
      provide: DYNAMODB_CLIENT,
      useFactory: (configService: ConfigService) => {
        return new DynamoDBClient({
          region: configService.get("AWS_REGION", "ap-southeast-1"),
          credentials: {
            accessKeyId: configService.get("AWS_ACCESS_KEY_ID"),
            secretAccessKey: configService.get("AWS_SECRET_ACCESS_KEY"),
          },
        });
      },
      inject: [ConfigService],
    },
    {
      provide: DYNAMODB_DOC_CLIENT,
      useFactory: (client: DynamoDBClient) => {
        return DynamoDBDocumentClient.from(client, {
          marshallOptions: {
            removeUndefinedValues: true,
          },
        });
      },
      inject: [DYNAMODB_CLIENT],
    },
  ],
  exports: [DYNAMODB_CLIENT, DYNAMODB_DOC_CLIENT],
})
export class DynamoDBModule {}
```

#### Repository Pattern

```typescript
// users.repository.ts
import { Injectable, Inject } from "@nestjs/common";
import { DynamoDBDocumentClient, GetCommand, PutCommand, QueryCommand } from "@aws-sdk/lib-dynamodb";
import { DYNAMODB_DOC_CLIENT } from "./dynamodb.module";

@Injectable()
export class UsersRepository {
  private readonly tableName = "MainTable";

  constructor(
    @Inject(DYNAMODB_DOC_CLIENT)
    private readonly docClient: DynamoDBDocumentClient,
  ) {}

  async findById(userId: string) {
    const command = new GetCommand({
      TableName: this.tableName,
      Key: {
        pk: `USER#${userId}`,
        sk: "PROFILE",
      },
    });

    const response = await this.docClient.send(command);
    return response.Item;
  }

  async create(user: CreateUserDto) {
    const item = {
      pk: `USER#${user.id}`,
      sk: "PROFILE",
      ...user,
      entityType: "User",
      createdAt: new Date().toISOString(),
    };

    const command = new PutCommand({
      TableName: this.tableName,
      Item: item,
      ConditionExpression: "attribute_not_exists(pk)",
    });

    await this.docClient.send(command);
    return item;
  }

  async findUserOrders(userId: string) {
    const command = new QueryCommand({
      TableName: this.tableName,
      KeyConditionExpression: "pk = :pk AND begins_with(sk, :skPrefix)",
      ExpressionAttributeValues: {
        ":pk": `USER#${userId}`,
        ":skPrefix": "ORDER#",
      },
    });

    const response = await this.docClient.send(command);
    return response.Items;
  }
}
```

#### Service Layer

```typescript
// users.service.ts
import { Injectable, NotFoundException } from "@nestjs/common";
import { UsersRepository } from "./users.repository";

@Injectable()
export class UsersService {
  constructor(private readonly usersRepository: UsersRepository) {}

  async getUser(userId: string) {
    const user = await this.usersRepository.findById(userId);
    if (!user) {
      throw new NotFoundException(`User ${userId} not found`);
    }
    return user;
  }

  async createUser(createUserDto: CreateUserDto) {
    return this.usersRepository.create(createUserDto);
  }

  async getUserWithOrders(userId: string) {
    const [user, orders] = await Promise.all([
      this.usersRepository.findById(userId),
      this.usersRepository.findUserOrders(userId),
    ]);

    if (!user) {
      throw new NotFoundException(`User ${userId} not found`);
    }

    return { ...user, orders };
  }
}
```

### 5.2. Sử dụng Dynamoose

#### Setup

```typescript
// dynamoose.module.ts
import { Module, OnModuleInit } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import * as dynamoose from "dynamoose";

@Module({})
export class DynamooseModule implements OnModuleInit {
  constructor(private configService: ConfigService) {}

  onModuleInit() {
    const ddb = new dynamoose.aws.ddb.DynamoDB({
      region: this.configService.get("AWS_REGION"),
      credentials: {
        accessKeyId: this.configService.get("AWS_ACCESS_KEY_ID"),
        secretAccessKey: this.configService.get("AWS_SECRET_ACCESS_KEY"),
      },
    });

    dynamoose.aws.ddb.set(ddb);
  }
}
```

#### Model Definition

```typescript
// user.model.ts
import * as dynamoose from "dynamoose";

const userSchema = new dynamoose.Schema({
  id: {
    type: String,
    hashKey: true,
  },
  name: {
    type: String,
    required: true,
  },
  email: {
    type: String,
    required: true,
    index: {
      name: "email-index",
      global: true,
    },
  },
  role: {
    type: String,
    enum: ["admin", "user"],
    default: "user",
  },
  createdAt: {
    type: Date,
    default: () => new Date(),
  },
});

export const UserModel = dynamoose.model("Users", userSchema);
```

#### Repository với Dynamoose

```typescript
// users.repository.ts
import { Injectable } from "@nestjs/common";
import { UserModel } from "./user.model";

@Injectable()
export class UsersRepository {
  async findById(id: string) {
    return UserModel.get(id);
  }

  async create(user: CreateUserDto) {
    return UserModel.create(user);
  }

  async findByEmail(email: string) {
    return UserModel.query("email").eq(email).using("email-index").exec();
  }

  async update(id: string, updates: Partial<CreateUserDto>) {
    return UserModel.update(id, updates);
  }

  async delete(id: string) {
    return UserModel.delete(id);
  }
}
```

### 5.3. Sử dụng nest-onetable

#### Module Registration

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { ConfigService } from "@nestjs/config";
import { OnetableModule } from "nest-onetable";

@Module({
  imports: [
    OnetableModule.register({
      useFactory(config: ConfigService, client: DynamoDBClient) {
        return {
          client,
          name: config.get("ONETABLE_NAME"),
          partial: true,
          schema: {
            format: "onetable:1.1.0",
            version: "0.0.1",
            indexes: {
              primary: { hash: "pk", sort: "sk" },
              GSI1: { hash: "GSI1PK", sort: "GSI1SK" },
            },
            models: {},
          },
        };
      },
      inject: [ConfigService, DynamoDBClient],
    }),
  ],
})
export class AppModule {}
```

#### Model với OneModel

```typescript
// users.model.ts
import { Injectable } from "@nestjs/common";
import { OneModel } from "nest-onetable";

@Injectable()
export class UsersModel extends OneModel("Users", {
  pk: { type: String, value: "USER#${userId}", hidden: true },
  sk: { type: String, value: "PROFILE", hidden: true },
  userId: { type: String, generate: "ulid", required: true },
  name: { type: String, required: true },
  email: { type: String, required: true },
  role: { type: String, enum: ["admin", "user"], default: "user" },
}) {}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(private readonly usersModel: UsersModel) {}

  async findAll() {
    return this.usersModel.find();
  }

  async findById(userId: string) {
    return this.usersModel.get({ userId });
  }

  async create(data: CreateUserDto) {
    return this.usersModel.create(data);
  }
}
```

---

## 6. Tài Liệu Tham Khảo

### Official Documentation
- [AWS DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
- [NestJS Documentation](https://docs.nestjs.com)

### Libraries
- [Dynamoose](https://dynamoosejs.com/)
- [nest-onetable](https://github.com/onhate/nest-onetable)
- [DynamoDB OneTable](https://doc.onetable.io/)
- [dynamodb-toolbox](https://github.com/jeremydaly/dynamodb-toolbox)

### Learning Resources
- [AWS re:Invent 2019: Data modeling with DynamoDB](https://www.youtube.com/watch?v=DIQVJqiSUkE)
- [The DynamoDB Book by Alex DeBrie](https://www.dynamodbbook.com/)
