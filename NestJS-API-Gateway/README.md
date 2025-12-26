# NestJS API Gateway - Best Practices 2025

> **Mục đích:** Hướng dẫn toàn diện về API Gateway patterns cho kiến trúc NestJS Microservices, cập nhật best practices 2024-2025.

---

## 📁 Cấu Trúc Tài Liệu

| File | Nội dung | Dành cho |
|------|----------|----------|
| `README.md` | Tổng quan & hướng dẫn đọc | Tất cả |
| `NestJS-API-Gateway.md` | Hướng dẫn chi tiết (Junior/Middle/Senior) | Junior → Senior |
| `Advanced-Patterns.md` | Performance, testing, custom patterns | Senior |
| `Principal-Level-Patterns.md` | Enterprise architecture, scaling | Principal/Staff |
| `RESEARCH_SUMMARY.md` | Tóm tắt nghiên cứu & nguồn tham khảo | Reference |

---

## 🎯 Cách Sử Dụng Tài Liệu

### Cho Junior Developer
1. Đọc `NestJS-API-Gateway.md` - phần **Junior Level**
2. Setup NestJS API Gateway cơ bản với TCP transport
3. Nắm vững: ClientProxy, MessagePattern, routing

### Cho Middle Developer
1. Đọc toàn bộ `NestJS-API-Gateway.md`
2. Tập trung: gRPC/Kafka transport, JWT Auth, Rate Limiting

### Cho Senior Developer
1. Đọc thêm `Advanced-Patterns.md`
2. Tập trung: Circuit Breaker, Distributed Tracing, Testing strategies

### Cho Principal/Staff
1. Đọc `Principal-Level-Patterns.md`
2. Tập trung: Multi-region, Migration, Enterprise patterns

---

## 🔑 Key Concepts

**API Gateway** trong NestJS microservices là service đóng vai trò "cổng vào duy nhất":

```
┌─────────────┐     ┌─────────────────────┐     ┌─────────────────┐
│   Clients   │────▶│ NestJS API Gateway  │────▶│ Microservices   │
│ (Web/Mobile)│     │  (HTTP → TCP/gRPC)  │     │ (NestJS/Others) │
└─────────────┘     └─────────────────────┘     └─────────────────┘
```

**Transports được hỗ trợ:** TCP, Redis, NATS, RabbitMQ, Kafka, gRPC

---

## 🚀 Quick Start

```bash
# Tạo NestJS API Gateway
nest new api-gateway

# Cài dependencies
cd api-gateway
npm install @nestjs/microservices
npm install @nestjs/throttler  # Rate limiting
```

---

## 📚 External Resources

- [NestJS Microservices Docs](https://docs.nestjs.com/microservices/basics)
- [NestJS 2025 Blueprint Podcast](https://www.linkedin.com/posts/anatolii-kosorukov-444a97109_podcast-1-the-2025-nestjs-blueprint-architecture-activity-7402888042308702208)
- [Microservices with NestJS, Kafka & Redis Tutorial](https://www.djamware.com/post/microservices-with-nestjs)
