# Advanced Patterns - DynamoDB với NestJS

Tài liệu này dành cho **Senior developers**, tập trung vào các patterns nâng cao.

---

## 1. Single Table Design Patterns

### 1.1. Entity Modeling

```typescript
// Key factories cho single table design
const keys = {
  user: {
    pk: (userId: string) => `USER#${userId}`,
    sk: () => "PROFILE",
  },
  order: {
    pk: (userId: string) => `USER#${userId}`,
    sk: (orderId: string) => `ORDER#${orderId}`,
  },
};

// Data layout:
// pk: USER#123, sk: PROFILE        -> User profile
// pk: USER#123, sk: ORDER#001      -> Order 1 của user 123
// pk: USER#123, sk: ORDER#002      -> Order 2 của user 123
```

### 1.2. One-to-Many với Query

```typescript
async getUserWithOrders(userId: string) {
  const command = new QueryCommand({
    TableName: "MainTable",
    KeyConditionExpression: "pk = :pk",
    ExpressionAttributeValues: { ":pk": `USER#${userId}` },
  });

  const items = (await docClient.send(command)).Items || [];
  return {
    profile: items.find(item => item.sk === "PROFILE"),
    orders: items.filter(item => item.sk.startsWith("ORDER#")),
  };
}
```

### 1.3. GSI Overloading

```typescript
// Dùng 1 GSI cho nhiều access patterns
// GSI1PK: STATUS#pending     -> All pending orders
// GSI1PK: EMAIL#john@x.com   -> User by email

const orderItem = {
  pk: `USER#${userId}`,
  sk: `ORDER#${orderId}`,
  GSI1PK: `STATUS#${status}`,
  GSI1SK: createdAt,
};

const userItem = {
  pk: `USER#${id}`,
  sk: "PROFILE",
  GSI1PK: `EMAIL#${email}`,
  GSI1SK: `USER#${id}`,
};
```

---

## 2. Advanced Query Patterns

### 2.1. Sparse Indexes

```typescript
// Chỉ index items cần thiết
async function createOrder(order: Order) {
  const item: any = { pk: `USER#${order.userId}`, sk: `ORDER#${order.id}`, ...order };
  
  // Chỉ priority orders mới có GSI
  if (order.isPriority) {
    item.GSI1PK = "PRIORITY";
    item.GSI1SK = order.createdAt;
  }
  
  await docClient.send(new PutCommand({ TableName: "MainTable", Item: item }));
}
```

### 2.2. Time-Based Queries

```typescript
// Partition by date để tránh hot partition
const item = {
  pk: `EVENT#${new Date().toISOString().split("T")[0]}`,  // EVENT#2024-01-01
  sk: `${timestamp}#${eventId}`,
  ttl: Math.floor(Date.now() / 1000) + (30 * 24 * 60 * 60),  // 30 days
};
```

---

## 3. Performance Optimization

### 3.1. Write Sharding

```typescript
// Tránh hot partitions với sharding
class HighTrafficCounter {
  private readonly shardCount = 10;

  async increment(counterId: string) {
    const shard = Math.floor(Math.random() * this.shardCount);
    await docClient.send(new UpdateCommand({
      TableName: "Counters",
      Key: { pk: `COUNTER#${counterId}`, sk: `SHARD#${shard}` },
      UpdateExpression: "ADD #count :amount",
      ExpressionAttributeNames: { "#count": "count" },
      ExpressionAttributeValues: { ":amount": 1 },
    }));
  }

  async getCount(counterId: string): Promise<number> {
    const response = await docClient.send(new QueryCommand({
      TableName: "Counters",
      KeyConditionExpression: "pk = :pk",
      ExpressionAttributeValues: { ":pk": `COUNTER#${counterId}` },
    }));
    return (response.Items || []).reduce((sum, s) => sum + (s.count || 0), 0);
  }
}
```

### 3.2. Batch Processing

```typescript
async function batchWrite(items: any[]) {
  const chunks = [];
  for (let i = 0; i < items.length; i += 25) {
    chunks.push(items.slice(i, i + 25));
  }

  for (const chunk of chunks) {
    const command = new BatchWriteCommand({
      RequestItems: {
        MainTable: chunk.map(item => ({ PutRequest: { Item: item } })),
      },
    });
    await docClient.send(command);
    await new Promise(r => setTimeout(r, 100));  // Throttle
  }
}
```

---

## 4. Testing với DynamoDB Local

### 4.1. Setup

```typescript
// docker run -p 8000:8000 amazon/dynamodb-local

function createLocalClient() {
  return new DynamoDBClient({
    endpoint: "http://localhost:8000",
    region: "local",
    credentials: { accessKeyId: "fake", secretAccessKey: "fake" },
  });
}
```

### 4.2. Repository Tests

```typescript
describe("UsersRepository", () => {
  let repository: UsersRepository;

  beforeAll(async () => {
    const client = createLocalClient();
    await createTestTable(client, "TestTable");
    // ... setup repository
  });

  it("should create a new user", async () => {
    const user = { id: "test-1", name: "Test", email: "test@x.com" };
    const result = await repository.create(user);
    expect(result).toMatchObject(user);
  });
});
```

---

## 5. Error Handling

### 5.1. Custom Exceptions

```typescript
export class ItemNotFoundException extends Error {
  constructor(key: string) {
    super(`Item with key ${key} not found`);
    this.name = "ItemNotFoundException";
  }
}
```

### 5.2. Retry Decorator

```typescript
function Retry(maxRetries = 3, baseDelay = 100) {
  return (target: any, key: string, desc: PropertyDescriptor) => {
    const original = desc.value;
    desc.value = async function (...args: any[]) {
      for (let i = 0; i <= maxRetries; i++) {
        try { return await original.apply(this, args); }
        catch (e: any) {
          if (e.name !== "ProvisionedThroughputExceededException" || i === maxRetries) throw e;
          await new Promise(r => setTimeout(r, baseDelay * Math.pow(2, i)));
        }
      }
    };
  };
}
```

---

## Tham Khảo

- [AWS DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [The DynamoDB Book](https://www.dynamodbbook.com/)
