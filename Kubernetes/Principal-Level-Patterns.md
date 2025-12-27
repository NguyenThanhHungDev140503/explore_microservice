# Principal Level Patterns (System Design & Enterprise)

## 1. Enterprise-Scale Architecture

### Multi-Tenancy & Isolation
Khi chạy nhiều team/dự án trên cùng hạ tầng K8s:
- **Soft Isolation**: Dùng **Namespaces** + **RBAC** + **Network Policies**. Phù hợp nội bộ công ty.
- **Hard Isolation**: Dùng **vCluster** (Virtual Clusters) hoặc dedicated Node Pools. Hoặc tách hẳn Cluster vật lý cho các môi trường nhạy cảm (Compliance, PCI-DSS).

### High Availability (HA) Control Plane
- Etcd phải chạy cluster (thường 3 hoặc 5 nodes) để đảm bảo dữ liệu không mất.
- Control Plane component (API Server, Scheduler) phải được replicate và load balance.
- Worker Nodes phải trải dài trên nhiều **Availability Zones (AZ)** (Multi-AZ) để chống disaster một Zone bị sập.

## 2. GitOps & Continuous Delivery
Chuyển dịch từ CI/CD push-based sang **GitOps pull-based**.
- **Công cụ**: ArgoCD hoặc FluxCD.
- **Nguyên lý**: 
    - Git là Single Source of Truth cho toàn bộ trạng thái Cluster.
    - ArgoCD agent chạy trong cluster, liên tục sync trạng thái từ Git vào K8s.
    - Không ai được phép `kubectl apply` thủ công.
    - Dễ dàng Audit, Rollback (chỉ cần `git revert`), Disaster Recovery.

## 3. Observability Stack (Review)
Không chỉ monitoring cơ bản, cần Full Observability:
- **Metrics**: Prometheus (thu thập) + Thanos/Cortex (lưu trữ lâu dài, Global view). Grafana (Dashboard).
- **Logs**: EFK Stack (Elasticsearch, Fluentd/Fluent-bit, Kibana) hoặc Loki (nhẹ hơn, tích hợp tốt Grafana).
- **Tracing**: Jaeger hoặc Tempo. Tích hợp OpenTelemetry trong code để trace request qua các microservices.

## 4. Service Mesh
Khi số lượng services quá lớn (>50), việc quản lý network, security, retry logic trở nên khó khăn.
- **Công cụ**: Istio, Linkerd.
- **Tính năng**:
    - **mTLS**: Mã hóa traffic giữa các services tự động.
    - **Traffic Splitting**: Canary deployment % traffic chính xác.
    - **Circuit Breaking & Retries**: Xử lý lỗi mạng thông minh mà không cần sửa code ứng dụng.
    - **Deep Observability**: Tự động vẽ bản đồ giao tiếp services (Service Map).
*Lưu ý: Service Mesh tăng độ phức tạp và latency, cần cân nhắc kỹ (Rule of thumb: Đừng dùng nếu chưa thực sự đau).*

## 5. Security Hardening (DevSecOps)
- **Pod Security Standards (PSS)**: Bắt buộc (Enforce) các rule như: không chạy container với root user (runAsNonRoot), không mount hostPath nhạy cảm, readonly filesystem.
- **Policy Engine**: Sử dụng **OPA Gatekeeper** hoặc **Kyverno** để viết policy as code. (Ví dụ: Từ chối deploy nếu Image không đến từ registry nội bộ).
- **Image Scanning**: Quét lỗ hổng image ngay trong CI và định kỳ trong Registry (Trivy, Clair).

## 6. Cost Optimization (FinOps)
- **Cluster Autoscaler**: Tự động tắt bật Node máy ảo. Dùng **Spot Instances** cho các workload chịu lỗi (stateless, batch job) để giảm tới 90% chi phí.
- **Cost Allocation**: Gắn Label (Cost Center, Team) vào resource để chia bill. Dùng **Kubecost** để soi chi phí từng namespace/deployment.

## 7. Migration Strategies (Legacy to Cloud Native)
- **Strangler Fig Pattern**: Tách dần functionality của Monolith thành Microservice chạy trên K8s, dùng Ingress để route traffic.
- **ExternalName Service**: Map Service trong K8s trỏ ra DB/API legacy bên ngoài.
