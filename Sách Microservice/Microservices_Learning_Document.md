# Tài Liệu Học Tập: Nền Tảng Microservices

Tài liệu này được tổng hợp dựa trên cuốn sách **"Building Microservices: Designing Fine-Grained Systems"** (Second Edition) của Sam Newman.

## 1. Monolith vs Microservices

### 1.1. Monolith (Kiến trúc Đơn khối)
Monolith là một đơn vị triển khai (deployment unit). Khi tất cả các chức năng của hệ thống phải được triển khai cùng nhau, đó được coi là một khối monolith.

Có ba dạng monolith chính:
1.  **Single-Process Monolith (Monolith đơn tiến trình):** Toàn bộ mã nguồn (code) được đóng gói vào một tiến trình duy nhất (process). Đây là dạng phổ biến nhất.
2.  **Modular Monolith (Monolith mô-đun hóa):** Một biến thể trong đó một tiến trình bao gồm các mô-đun riêng biệt. Dù các mô-đun có thể được phát triển độc lập, chúng vẫn phải được triển khai cùng nhau.
3.  **Distributed Monolith (Monolith phân tán):** Hệ thống bao gồm nhiều dịch vụ (services), nhưng toàn bộ hệ thống phải được triển khai cùng nhau. Nó thường mang lại nhược điểm của cả hệ thống phân tán và monolith mà không có nhiều lợi ích.

**Ưu điểm của Monolith:**
*   Quy trình triển khai (deployment topology) đơn giản hơn.
*   Dễ dàng phát triển, giám sát (monitoring) và gỡ lỗi (troubleshooting) hơn so với hệ thống phân tán.
*   Tái sử dụng code trong nội bộ đơn giản hơn.
*   Là lựa chọn mặc định hợp lý cho nhiều tổ chức, đặc biệt là giai đoạn đầu (startup).

### 1.2. Microservices
Microservices là các dịch vụ có thể phát hành độc lập (independently releasable services), được mô hình hóa xoay quanh các nghiệp vụ kinh doanh (business domain).

**Các đặc điểm chính:**
*   **Triển khai độc lập (Independent Deployability):** Có thể thay đổi và triển khai một dịch vụ mà không cần triển khai lại các dịch vụ khác.
*   **Mô hình hóa theo nghiệp vụ (Modeled Around Business Domain):** Dịch vụ đại diện cho các chức năng nghiệp vụ, giúp dễ dàng thay đổi và triển khai các tính năng mới.
*   **Sở hữu trạng thái riêng (Owning Their Own State):** Microservices tránh sử dụng database chia sẻ (shared database). Mỗi dịch vụ nên đóng gói dữ liệu của riêng mình.
*   **Che giấu thông tin (Information Hiding):** Chi tiết triển khai bên trong được ẩn đi, chỉ lộ ra các giao diện (interface) cần thiết.

**Ưu điểm của Microservices:**
*   **Sự đa dạng công nghệ (Technology Heterogeneity):** Có thể sử dụng các công nghệ khác nhau cho từng dịch vụ phù hợp nhất.
*   **Khả năng phục hồi (Robustness):** Lỗi ở một dịch vụ có thể được cô lập (bulkhead), không làm sập toàn bộ hệ thống.
*   **Mở rộng (Scaling):** Có thể mở rộng (scale) từng dịch vụ riêng lẻ thay vì cả hệ thống.
*   **Dễ dàng triển khai (Ease of Deployment):** Triển khai các thay đổi nhỏ nhanh chóng hơn và ít rủi ro hơn.
*   **Tương thích tổ chức (Organizational Alignment):** Phù hợp với các nhóm nhỏ, làm việc độc lập.

---

## 2. Loose Coupling & High Cohesion (Kết nối lỏng & Tính gắn kết cao)

Để đạt được khả năng triển khai độc lập, kiến trúc Microservices cần đảm bảo hai nguyên tắc cốt lõi: **Loose Coupling** và **High Cohesion**.

### 2.1. High Cohesion (Tính gắn kết cao)
**Định nghĩa:** "Mã nguồn thay đổi cùng nhau thì nên ở cùng nhau" (The code that changes together, stays together).

*   Chúng ta muốn nhóm các hành vi (behavior) liên quan lại với nhau.
*   Nếu chức năng liên quan nằm rải rác khắp hệ thống, việc thay đổi sẽ rất tốn kém và rủi ro vì phải sửa đổi và triển khai nhiều dịch vụ cùng lúc.
*   Mục tiêu là **Strong Cohesion** (Gắn kết mạnh) trong phạm vi biên của dịch vụ.

### 2.2. Loose Coupling (Kết nối lỏng lẻo)
**Định nghĩa:** Thay đổi ở dịch vụ này không nên yêu cầu thay đổi ở dịch vụ khác.

*   Một dịch vụ nên biết càng ít càng tốt về các dịch vụ mà nó hợp tác.
*   Tránh các kiểu tích hợp chặt chẽ (tight coupling) khiến thay đổi ở một nơi làm hỏng các nơi khác.

**Các loại Coupling (từ lỏng đến chặt):**
1.  **Domain Coupling (Lỏng - Chấp nhận được):** Một dịch vụ cần sử dụng chức năng của dịch vụ khác để hoàn thành nhiệm vụ. Đây là điều không thể tránh khỏi nhưng cần tối thiểu hóa.
2.  **Pass-Through Coupling:** Một dịch vụ truyền dữ liệu cho một dịch vụ khác chỉ vì một dịch vụ thứ ba cần nó. Điều này làm lộ chi tiết triển khai và gây khó khăn khi thay đổi dữ liệu.
3.  **Common Coupling:** Hai hoặc nhiều dịch vụ dùng chung một nguồn dữ liệu (ví dụ: Shared Database). Thay đổi cấu trúc dữ liệu sẽ ảnh hưởng tất cả các dịch vụ.
4.  **Content Coupling (Chặt - Nên tránh):** Một dịch vụ truy cập trực tiếp và thay đổi dữ liệu bên trong của dịch vụ khác. Đây là loại coupling tồi tệ nhất vì phá vỡ nguyên tắc che giấu thông tin.

### 2.3. Mối quan hệ
*   **Định luật Constantine:** Cấu trúc ổn định nếu Cohesion cao và Coupling thấp.
*   Cohesion nói về mối quan hệ **bên trong** biên giới dịch vụ.
*   Coupling nói về mối quan hệ **giữa** các dịch vụ qua biên giới đó.

---

## 3. DDD Cơ bản: Bounded Context (Ngữ cảnh giới hạn)

**Domain-Driven Design (DDD)** là phương pháp chính để tìm ra biên giới (boundaries) của các microservices.

### 3.1. Các khái niệm cốt lõi của DDD
*   **Ubiquitous Language (Ngôn ngữ chung):** Sử dụng các thuật ngữ giống nhau trong cả mã nguồn (code) và giao tiếp nghiệp vụ để tránh hiểu nhầm.
*   **Aggregate (Hợp nhất):** Một tập hợp các đối tượng được quản lý như một đơn vị duy nhất, thường đại diện cho khái niệm thực tế (ví dụ: Đơn hàng, Hóa đơn). Aggregate có vòng đời và thay đổi trạng thái.

### 3.2. Bounded Context (Ngữ cảnh giới hạn)
**Bounded Context** là một ranh giới rõ ràng (explicit boundary) trong một miền nghiệp vụ (business domain).

*   **Đại diện cho ranh giới tổ chức:** Bounded Context thường tương ứng với một bộ phận hoặc chức năng nghiệm vụ lớn (ví dụ: Kho hàng, Tài chính).
*   **Che giấu chi tiết (Hidden Models):** Bên trong Bounded Context có các mô hình và chi tiết chỉ có ý nghĩa nội bộ (ví dụ: "Xe nâng" chỉ quan trọng với Kho hàng, không quan trọng với Tài chính).
*   **Giao diện rõ ràng:** Nó cung cấp chức năng cho hệ thống bên ngoài thông qua các giao diện (interface) rõ ràng, trong khi che giấu độ phức tạp bên trong.
*   **Shared Models (Mô hình chia sẻ):** Một khái niệm (như "Khách hàng") có thể tồn tại ở nhiều Bounded Context khác nhau nhưng với ý nghĩa và dữ liệu khác nhau (Tài chính quan tâm đến thanh toán của khách, Kho hàng quan tâm đến địa chỉ giao hàng của khách).

### 3.3. Áp dụng vào Microservices
*   Bounded Context là đơn vị lý tưởng để xác định biên giới của một Microservice.
*   Ban đầu, một Microservice có thể bao trùm cả một Bounded Context lớn.
*   Về sau, Bounded Context lớn có thể chứa các Bounded Context con (nested contexts), cho phép chia nhỏ thành các microservices nhỏ hơn (ví dụ: "Kho hàng" chia thành "Quản lý tồn kho" và "Vận chuyển"), nhưng vẫn có thể ẩn việc chia nhỏ này dưới một API chung.
