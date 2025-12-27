# Giáo trình Chuyên sâu: Các Pattern Cốt lõi trong Microservices

**Tài liệu tham khảo**: *Microservices Patterns: With examples in Java* - Chris Richardson

---

## Mục lục
1. [Pattern 1: Database per Service (Tách Biệt Cơ Sở Dữ Liệu)](#1-pattern-database-per-service)
2. [Pattern 2: Distributed Transactions với Saga Pattern](#2-distributed-transactions-với-saga-pattern)
3. [Pattern 3: Nâng cao Performance với CQRS](#3-nâng-cao-performance-với-cqrs)

---

## 1. Pattern: Database per Service

### 1.1 Khái niệm
Trong kiến trúc Monolith truyền thống, các module thường chia sẻ cùng một cơ sở dữ liệu (Shared Database). Điều này giúp việc quản lý transaction đơn giản (ACID) nhưng lại tạo ra sự phụ thuộc chặt chẽ (tight coupling) giữa các module.

**Database per Service** là pattern yêu cầu mỗi microservice phải có cơ sở dữ liệu riêng của nó. "Database" ở đây có thể hiểu là:
- Một schema riêng trong cùng một database server.
- Một database server riêng biệt.
- Các loại database khác nhau (Polyglot Persistence) phù hợp với nhu cầu từng service (VD: Order Service dùng MySQL, Search Service dùng Elasticsearch).

### 1.2 Tại sao cần tách Database?
*   **Loose Coupling (Giảm sự phụ thuộc)**: Các service có thể thay đổi cấu trúc dữ liệu mà không ảnh hưởng đến service khác.
*   **Independent Scaling (Scale độc lập)**: Service nào chịu tải cao có thể scale database của riêng nó.
*   **Công nghệ phù hợp**: Mỗi team có thể chọn công nghệ DB tốt nhất cho bài toán của mình (SQL vs NoSQL).

### 1.3 Thách thức & Giải pháp
Khi tách DB, chúng ta đánh mất các tiện ích của ACID transaction trên toàn hệ thống và khả năng join bảng đơn giản.
*   **Thách thức 1: Các Transaction trải rộng nhiều service (Distributed Transactions)** -> *Giải pháp: Dùng Saga Pattern (Xem phần 2).*
*   **Thách thức 2: Các truy vấn tổng hợp dữ liệu từ nhiều service (Distributed Queries)** -> *Giải pháp: Dùng API Composition hoặc CQRS (Xem phần 3).*

---

## 2. Distributed Transactions với Saga Pattern

### 2.1 Vấn đề của Distributed Transactions (2PC)
Các giao dịch phân tán truyền thống (như 2 Phase Commit - 2PC/XA) đảm bảo tính nhất quán mạnh (Strong Consistency) nhưng lại có nhược điểm lớn:
*   **Synchronous (Đồng bộ)**: Tất cả các bên tham gia phải online và chờ đợi nhau, giảm availability của toàn hệ thống.
*   **Locking**: Khóa dữ liệu trong thời gian dài chờ commit.
*   **Không tương thích**: Nhiều công nghệ hiện đại (NoSQL, Message/Event Brokers như Kafka) không hỗ trợ XA.

### 2.2 Saga Pattern là gì?
**Saga** là một chuỗi các **local transactions**. Mỗi local transaction cập nhật dữ liệu trong một service và publish một message/event để kích hoạt local transaction tiếp theo trong chuỗi.

*   Nếu một local transaction thất bại (do lỗi logic nghiệp vụ), Saga sẽ thực thi một chuỗi các **Compensating Transactions (Giao dịch bù)** để hoàn tác (undo) các thay đổi đã được thực hiện bởi các local transaction trước đó.

### 2.3 Cấu trúc của một Saga
Ví dụ quy trình `Create Order`:
1.  **Order Service**: Tạo Order (Trạng thái: PENDING) -> *Thành công*
2.  **Consumer Service**: Xác minh hạn mức tín dụng -> *Thành công*
3.  **Kitchen Service**: Xác minh nhà bếp -> *Thất bại! (Hết nguyên liệu)*
    *   *Kích hoạt quy trình bù:*
    *   **Consumer Service**: Hoàn lại hạn mức (nếu đã trừ).
    *   **Order Service**: Cập nhật trạng thái Order thành REJECTED.

### 2.4 Hai cách điều phối Saga (Coordination)

#### A. Choreography (Vũ điệu - Phi tập trung)
Các service tự giao tiếp với nhau qua Events mà không cần người chỉ huy.
*   **Cách hoạt động**:
    *   Order Service làm xong việc -> Bắn event `OrderCreated`.
    *   Customer Service lắng nghe `OrderCreated` -> Trừ tiền -> Bắn event `CreditReserved`.
    *   Kitchen Service lắng nghe `CreditReserved` -> ...
*   **Ưu điểm**: Đơn giản, loose coupling.
*   **Nhược điểm**: Khó theo dõi quy trình phức tạp, nguy cơ tạo ra "vòng lặp" dependency.

#### B. Orchestration (Nhạc trưởng - Tập trung)
Sử dụng một class hoặc service trung tâm (Saga Orchestrator) để điều phối.
*   **Cách hoạt động**:
    *   `CreateOrderSaga` nhận lệnh.
    *   Gửi command `ReserveCredit` tới Customer Service.
    *   Nhận reply, gửi command `CreateTicket` tới Kitchen Service.
*   **Ưu điểm**: Dễ quản lý logic phức tạp, tránh phụ thuộc chéo giữa các service.
*   **Nhược điểm**: Cần thêm code quản lý Orchestrator.

### 2.5 Tính chất của Saga: ACD (Thiếu Isolation)
Saga tuân thủ Atomicity (tất cả hoặc không gì cả - thông qua bù), Consistency, Durability, nhưng thiếu **Isolation**.
*   Các thay đổi của local transaction được commit ngay lập tức và các saga khác có thể nhìn thấy (dirty reads).
*   Cần các biện pháp đối phó (Countermeasures) như Semantic Lock, Commutative updates, hoặc Versioning để xử lý các vấn đề về concurrency.

---

## 3. Nâng cao Performance với CQRS

### 3.1 Vấn đề của việc truy vấn trong Microservices
Với Database per Service, việc thực hiện các câu query phức tạp "Join" dữ liệu từ nhiều service (ví dụ: "Tìm đơn hàng của khách hàng tên Hùng có món Pizza") là rất khó.
*   **API Composition**: Gọi API từng service rồi ghép data lại. -> *Chậm, tốn tài nguyên, không filter/sort hiệu quả được.*

### 3.2 CQRS Pattern (Command Query Responsibility Segregation)
CQRS tách biệt trách nhiệm Ghi (Command) và Đọc (Query) ra làm hai phần riêng biệt.

#### Mô hình:
1.  **Command Side (Write Model)**:
    *   Xử lý các lệnh Create, Update, Delete.
    *   Thực hiện logic nghiệp vụ phức tạp.
    *   Sử dụng database tối ưu cho việc ghi và toàn vẹn dữ liệu (thường là Relational DB).
    *   Khi data thay đổi -> **Publish Domain Events**.

2.  **Query Side (Read Model)**:
    *   Xử lý các truy vấn GET.
    *   Sử dụng database tối ưu cho việc đọc (VD: Elasticsearch cho search, MongoDB cho document retrieval, hoặc Redis cho caching).
    *   **Subscribe** các Domain Events từ Command Side để cập nhật dữ liệu của mình (Data Synchronization).

#### Ví dụ: `Order History Service` (View Query)
*   **Nhu cầu**: Hiển thị lịch sử đơn hàng kèm chi tiết món ăn và tình trạng giao hàng.
*   **Thực hiện**:
    *   Đây là một Service riêng (hoặc một module Query riêng).
    *   Nó lắng nghe các events: `OrderCreated`, `DeliveryStarted`, `TicketAccepted`.
    *   Nó lưu trữ dữ liệu vào một bảng `OrderHistoryView` đã được Denormalized (phi chuẩn hóa) để khi query chỉ cần `SELECT * FROM OrderHistoryView WHERE userId = ...` là có đủ hết thông tin, cực nhanh.

### 3.3 Lợi ích & Hạn chế
*   **Lợi ích**:
    *   **Performance cực cao**: Tách biệt tải đọc/ghi, database đọc được tối ưu hóa hoàn toàn cho query.
    *   **Scalability**: Có thể scale số lượng instance phục vụ đọc độc lập với ghi.
    *   **Simplicity**: Query model đơn giản hơn do không chứa logic nghiệp vụ phức tạp.
*   **Hạn chế**:
    *   **Complexity**: Hệ thống phức tạp hơn, cần quản lý event synchronization.
    *   **Eventual Consistency (Tính nhất quán cuối cùng)**: Dữ liệu bên Query side sẽ có độ trễ nhất định so với Command side (thường tính bằng mili-giây hoặc giây). Hệ thống UI cần xử lý việc này (VD: Hiển thị trạng thái "đang cập nhật").
