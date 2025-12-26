# DynamoDB với NestJS

Bộ tài liệu nghiên cứu về Amazon DynamoDB và cách tích hợp với NestJS.

## Cấu Trúc Tài Liệu

| File | Mô tả | Đối tượng |
|------|-------|-----------|
| [DynamoDB.md](./DynamoDB.md) | Hướng dẫn toàn diện từ cơ bản đến nâng cao | Junior/Middle/Senior |
| [Advanced-Patterns.md](./Advanced-Patterns.md) | Patterns và optimization nâng cao | Senior |
| [Principal-Level-Patterns.md](./Principal-Level-Patterns.md) | Enterprise patterns và system design | Principal/Staff |
| [RESEARCH_SUMMARY.md](./RESEARCH_SUMMARY.md) | Tóm tắt nghiên cứu và nguồn tham khảo | All |

## Cách Sử Dụng Tài Liệu

### Cho Junior Developer
1. Đọc `DynamoDB.md` phần **Junior Level – Cơ Bản**
2. Hiểu các concepts cốt lõi: Partition Key, Sort Key, Table, Item
3. Thực hành với các examples cơ bản

### Cho Middle Developer
1. Đọc toàn bộ `DynamoDB.md` phần Junior và **Middle Level**
2. Tìm hiểu về Secondary Indexes (GSI/LSI)
3. Hiểu query patterns và data modeling

### Cho Senior Developer
1. Đọc `DynamoDB.md` đầy đủ
2. Đọc `Advanced-Patterns.md` để hiểu về performance optimization
3. Áp dụng Single Table Design patterns

### Cho Principal/Staff Engineer
1. Đọc toàn bộ các file trên
2. Focus vào `Principal-Level-Patterns.md` cho enterprise patterns
3. Nghiên cứu migration strategies và monitoring

## Key Concepts

### DynamoDB là gì?
Amazon DynamoDB là một **fully managed NoSQL database service** của AWS, cung cấp:
- **Serverless**: Không cần quản lý infrastructure
- **Performance at scale**: Single-digit millisecond latency
- **High availability**: 99.999% với Global Tables
- **Pay-per-use**: Chỉ trả tiền cho những gì sử dụng

### Primary Key Types
```
┌─────────────────────────────────────────────────────────┐
│                    PRIMARY KEY                          │
├─────────────────────────────────────────────────────────┤
│ Simple Primary Key    │ Composite Primary Key           │
│ (Partition Key only)  │ (Partition Key + Sort Key)      │
│                       │                                 │
│ pk: "USER#123"        │ pk: "USER#123"                  │
│                       │ sk: "PROFILE#info"              │
└─────────────────────────────────────────────────────────┘
```

### NestJS Integration Options
1. **AWS SDK v3**: `@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`
2. **Dynamoose**: ORM-like, Mongoose syntax
3. **nest-onetable**: NestJS module cho DynamoDB OneTable

## Quick Start

### Cài đặt dependencies

```bash
# AWS SDK v3 (recommended)
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb

# Hoặc Dynamoose (ORM-like)
npm install dynamoose

# Hoặc nest-onetable (NestJS module)
npm install nest-onetable dynamodb-onetable @aws-sdk/client-dynamodb
```

### Basic Setup với AWS SDK v3

```typescript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand, GetCommand } from "@aws-sdk/lib-dynamodb";

// Tạo DynamoDB client
const client = new DynamoDBClient({ region: "ap-southeast-1" });
const docClient = DynamoDBDocumentClient.from(client);

// Put item
await docClient.send(new PutCommand({
  TableName: "Users",
  Item: { id: "user-123", name: "John Doe", email: "john@example.com" }
}));

// Get item
const result = await docClient.send(new GetCommand({
  TableName: "Users",
  Key: { id: "user-123" }
}));
```

## External Resources

- [AWS DynamoDB Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
- [Dynamoose Documentation](https://dynamoosejs.com/)
- [nest-onetable GitHub](https://github.com/onhate/nest-onetable)
- [DynamoDB OneTable](https://doc.onetable.io/)
