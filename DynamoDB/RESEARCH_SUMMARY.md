# Research Summary: DynamoDB với NestJS

**Ngày nghiên cứu**: 26/12/2025  
**Mục đích**: Tìm hiểu DynamoDB và cách tích hợp với NestJS microservices

---

## 1. Mục Tiêu Research

### Vì sao nghiên cứu?
- DynamoDB là NoSQL database phổ biến của AWS, phù hợp cho microservices cần high performance và scalability
- Cần hiểu cách tích hợp DynamoDB vào NestJS projects một cách hiệu quả
- Tìm hiểu best practices và patterns để tránh common mistakes

### Dùng trong project nào?
- Microservices architecture với NestJS
- Applications cần low latency và high throughput
- Serverless applications (Lambda + DynamoDB)

---

## 2. Nguồn Thông Tin Đã Sử Dụng

### Official Documentation
| Nguồn | URL | Ngày truy cập | Ghi chú |
|-------|-----|---------------|---------|
| AWS DynamoDB Developer Guide | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ | 26/12/2025 | Tài liệu chính thức, đầy đủ nhất |
| AWS SDK for JavaScript v3 | https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/ | 26/12/2025 | SDK chính thức cho Node.js |
| NestJS Documentation | https://docs.nestjs.com | 26/12/2025 | Framework documentation |

### Third-party Libraries
| Nguồn | URL | Ngày truy cập | Ghi chú |
|-------|-----|---------------|---------|
| Dynamoose | https://dynamoosejs.com | 26/12/2025 | ORM-like cho DynamoDB, Mongoose syntax |
| nest-onetable | https://github.com/onhate/nest-onetable | 26/12/2025 | NestJS module cho DynamoDB OneTable |
| DynamoDB OneTable | https://doc.onetable.io/ | 26/12/2025 | Single table design library |
| dynamodb-toolbox | https://www.npmjs.com/package/dynamodb-toolbox | 26/12/2025 | Type-safe query builder |

### NPM Packages
| Package | Version | Mô tả |
|---------|---------|-------|
| @aws-sdk/client-dynamodb | 3.958.0 | Low-level DynamoDB client |
| @aws-sdk/lib-dynamodb | 3.958.0 | Document client (high-level) |
| dynamoose | 4.x | ORM-like modeling tool |
| nest-onetable | latest | NestJS integration với OneTable |

### GitHub Repositories
| Repo | URL | Ghi chú |
|------|-----|---------|
| onhate/nest-onetable | https://github.com/onhate/nest-onetable | NestJS DynamoDB OneTable integration |
| ocoda/event-sourcing | https://github.com/ocoda/event-sourcing | Event sourcing với DynamoDB |

---

## 3. Phát Hiện Chính (Key Findings)

### Ưu điểm của DynamoDB

1. **Serverless & Fully Managed**
   - Không cần quản lý infrastructure
   - Auto scaling capacity
   - Zero maintenance downtime

2. **Performance**
   - Single-digit millisecond latency
   - Consistent performance at any scale
   - Warm throughput prevents cold starts

3. **High Availability**
   - 99.999% SLA với Global Tables
   - Multi-region replication
   - Zero RPO (Recovery Point Objective)

4. **Cost Optimization**
   - Pay-per-request pricing
   - Provisioned capacity option
   - On-demand scaling

### Hạn chế của DynamoDB

1. **Query Flexibility**
   - Không hỗ trợ SQL-like queries
   - Cần design schema dựa trên access patterns
   - Join operations phức tạp

2. **Learning Curve**
   - Single table design cần kinh nghiệm
   - Partition key design quan trọng
   - GSI/LSI limits và costs

3. **Data Types**
   - Limited data types so với SQL databases
   - Maximum item size: 400KB
   - Nested attributes có depth limits

### So sánh các thư viện NestJS

| Thư viện | Ưu điểm | Nhược điểm | Use case |
|----------|---------|------------|----------|
| AWS SDK v3 | Full control, official | Verbose, low-level | Production, full control |
| Dynamoose | Mongoose-like, easy | Abstraction overhead | Rapid development |
| nest-onetable | NestJS integration | Newer, smaller community | Single table design |

---

## 4. Kiến Trúc/Cách Dùng Đề Xuất

### Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     NestJS Application                       │
├─────────────────────────────────────────────────────────────┤
│  Controller Layer                                            │
│  ├── UsersController                                         │
│  ├── OrdersController                                        │
│  └── ProductsController                                      │
├─────────────────────────────────────────────────────────────┤
│  Service Layer                                               │
│  ├── UsersService                                            │
│  ├── OrdersService                                           │
│  └── ProductsService                                         │
├─────────────────────────────────────────────────────────────┤
│  Repository Layer (DynamoDB Access)                          │
│  ├── UsersRepository                                         │
│  ├── OrdersRepository                                        │
│  └── ProductsRepository                                      │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure Layer                                        │
│  ├── DynamoDBModule (Configuration)                          │
│  ├── DynamoDBClient (AWS SDK v3)                            │
│  └── Document Client (lib-dynamodb)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
               ┌─────────────────────────────┐
               │      Amazon DynamoDB         │
               │                             │
               │  ┌─────────────────────┐    │
               │  │   Single Table      │    │
               │  │   - Users           │    │
               │  │   - Orders          │    │
               │  │   - Products        │    │
               │  └─────────────────────┘    │
               │                             │
               │  ┌─────────────────────┐    │
               │  │   GSI (indexes)     │    │
               │  └─────────────────────┘    │
               └─────────────────────────────┘
```

### Patterns nên tránh

1. ❌ **Không scan toàn bộ table** - Rất tốn RCU và chậm
2. ❌ **Không sử dụng hot partitions** - Cần distribute workload đều
3. ❌ **Không lưu large items** - Maximum 400KB/item
4. ❌ **Không design schema trước khi biết access patterns**

### Patterns nên áp dụng

1. ✅ **Single Table Design** - Một table cho nhiều entities
2. ✅ **Composite Sort Keys** - Cho hierarchical data
3. ✅ **GSI Overloading** - Tận dụng tối đa 20 GSI
4. ✅ **Sparse Indexes** - Chỉ index items cần thiết
5. ✅ **Write Sharding** - Cho high-traffic partitions

---

## 5. Use Cases Đã Xác Định

| Use Case | File tham khảo | Mô tả |
|----------|----------------|-------|
| Basic CRUD operations | DynamoDB.md (Junior Level) | GetItem, PutItem, UpdateItem, DeleteItem |
| Query with sort key | DynamoDB.md (Middle Level) | Query với begins_with, between |
| Pagination | DynamoDB.md (Middle Level) | LastEvaluatedKey, ExclusiveStartKey |
| Secondary Indexes | DynamoDB.md (Senior Level) | GSI/LSI design và usage |
| Single Table Design | Advanced-Patterns.md | Entity modeling trong single table |
| Transaction operations | Advanced-Patterns.md | TransactWriteItems, TransactGetItems |
| Event Sourcing | Principal-Level-Patterns.md | DynamoDB Streams + Event Store |
| Global Tables | Principal-Level-Patterns.md | Multi-region deployment |

---

## 6. Các Quyết Định Quan Trọng

### Chọn SDK/Library

**Quyết định**: Sử dụng **AWS SDK v3** (`@aws-sdk/lib-dynamodb`) làm default choice.

**Lý do**:
- Official AWS support
- Full control over operations
- Type-safe với TypeScript
- Modular, tree-shakeable
- Better performance

**Alternative**: Sử dụng `dynamoose` cho rapid prototyping hoặc projects cần Mongoose-like syntax.

### Table Design Strategy

**Quyết định**: Áp dụng **Single Table Design** cho production applications.

**Lý do**:
- Reduced operational overhead
- Better performance (1 request vs multiple)
- Cost optimization
- Simplified backup/restore

---

## 7. Next Steps

1. [ ] Đọc chi tiết `DynamoDB.md` để hiểu core concepts
2. [ ] Thực hành với AWS SDK v3 examples
3. [ ] Thiết kế schema cho specific use case
4. [ ] Implement repository pattern trong NestJS
5. [ ] Test với DynamoDB Local trước khi deploy
