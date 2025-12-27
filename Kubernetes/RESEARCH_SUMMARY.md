# Research Summary: Kubernetes

## 1. Mục Tiêu Research
- **Mục đích**: Tìm hiểu sâu về Kubernetes (K8s) để ứng dụng trong việc triển khai, quản lý và scale các ứng dụng microservices.
- **Project**: `explore_microservice`.
- **Goals**:
    - Hiểu kiến trúc cốt lõi và các concepts cơ bản.
    - Nắm vững các patterns nâng cao để tối ưu hóa hiệu năng và chi phí.
    - Xây dựng chiến lược triển khai enterprise-level (High Availability, Security, GitOps).

## 2. Nguồn Thông Tin Đã Sử Dụng

### Official Documentation & Community
- **Kubernetes Official Docs**: [kubernetes.io](https://kubernetes.io/docs/home/) - Nguồn chính thống về concepts và API.
- **CNCF (Cloud Native Computing Foundation)**: [cncf.io](https://www.cncf.io/) - Thông tin về hệ sinh thái Cloud Native.

### Blogs & Articles
- **Kubernetes Architecture**: [Cyberlab](https://cyberlab.rs), [RedHat](https://www.redhat.com) - Giải thích chi tiết về Control Plane và Worker Nodes.
- **Advanced Patterns**: [Medium - Advanced K8s Patterns](https://medium.com), [Cast.ai](https://cast.ai) - Các bài viết về Autoscaling (HPA/VPA) và Cost Optimization.
- **Comparisons**: [Kubernetes vs Docker Swarm vs Nomad 2024](https://best100tools.com) - So sánh ưu nhược điểm để chọn lựa giải pháp phù hợp.

## 3. Phát Hiện Chính (Key Findings)

### Ưu điểm của Kubernetes
- **Scalability**: Khả năng mở rộng vượt trội cho các hệ thống lớn, hỗ trợ HPA, VPA và Cluster Autoscaler.
- **Ecosystem**: Hệ sinh thái khổng lồ với Helm, Prometheus, Istio, ArgoCD, v.v.
- **Portability**: Chạy được trên mọi môi trường (On-premise, Hybrid, Multi-cloud).
- **Self-healing**: Tự động restart pods lỗi, thay thế node chết.

### Hạn chế
- **Domplexity**: Learning curve dốc, cấu hình phức tạp (Networking, Storage, Security).
- **Resource Heaviness**: Yêu cầu tài nguyên hệ thống cơ bản cao hơn so với Docker Swarm hay Nomad.

### So sánh
- **vs Docker Swarm**: Swarm đơn giản, dễ setup nhưng thiếu các tính năng nâng cao và automation mạnh mẽ như K8s.
- **vs Nomad**: Nomad linh hoạt (chạy cả non-container workloads) và đơn giản hơn, nhưng cộng đồng nhỏ hơn K8s.

## 4. Kiến Trúc/Cách Dùng Đề Xuất

### Mô hình đề xuất cho `explore_microservice`
- **Development**: Sử dụng **Minikube** hoặc **Kind** để chạy cluster local nhẹ nhàng.
- **Production**:
    - **Control Plane**: Managed Kubernetes (EKS/GKE/AKS) để giảm tải vận hành.
    - **GitOps**: Sử dụng **ArgoCD** để quản lý deployments đồng bộ với Git.
    - **Ingress Controller**: **Nginx Ingress** cho traffic routing cơ bản.
    - **Monitoring**: Stack **Prometheus + Grafana** để giám sát resource và application metrics.

### Patterns nên tránh
- **"ClickOps"**: Tránh thay đổi cấu hình trực tiếp trên cluster (kubectl edit), luôn thông qua Git (IaC).
- **Naked Pods**: Không bao giờ chạy Pod lẻ mà không có Deployment/ReplicaSet quản lý.
- **No Resource Limits**: Luôn set requests/limits để tránh việc một pod chiếm dụng toàn bộ tài nguyên node.

## 5. Use Cases Đã Xác Định

- **Microservices Deployment**: Quản lý hàng chục services nhỏ lẻ, scale độc lập.
- **Zero-downtime Deployment**: Rolling Updates, Canary Deployments.
- **Stateful Applications**: Database cluster (mặc dù nên dùng managed DB, nhưng K8s vẫn hỗ trợ qua StatefulSet).
- **Batch Processing**: CronJobs cho các tác vụ định kỳ.
