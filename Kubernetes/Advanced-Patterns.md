# Advanced Patterns (Senior Level)

## 1. Advanced Resource Management

### Resource Requests & Limits
Đây là yếu tố **quan trọng nhất** để đảm bảo cluster ổn định (Quality of Service - QoS).
- **Requests**: Số resource tối thiểu node phải có để Pod được schedule.
- **Limits**: Giới hạn tối đa Pod được dùng. Nếu vượt Memory limit -> OOMKilled; vượt CPU limit -> Throttling.

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m" # 0.25 core
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### LimitRanges & ResourceQuotas
- **LimitRange**: Thiết lập default request/limit cho Pod trong 1 Namespace nếu user quên set.
- **ResourceQuota**: Giới hạn tổng resource của 1 Namespace (tránh team này ăn hết tài nguyên team kia).

## 2. Scheduling Cao Cấp

### Affinity & Anti-Affinity
Điều khiển việc Pod nằm trên Node nào (Node Affinity) hoặc nằm cùng/khác Node với Pod khác (Pod Affinity/Anti-Affinity).

Example: Đảm bảo 3 replicas của app API không bao giờ nằm chung 1 node (để High Availability).
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - my-api
        topologyKey: "kubernetes.io/hostname"
```

### Taints & Tolerations
- **Taint** (đặt ở Node): "Đánh dấu" Node không cho Pod thường nhảy vào (ví dụ: node cho GPU, node cho Master).
- **Toleration** (đặt ở Pod): Permisstion để Pod được nhảy vào Node có Taint tương ứng.

## 3. Autoscaling Strategies

### Horizontal Pod Autoscaler (HPA)
Tự động tăng giảm số lượng Replicas dựa trên CPU/Memory hoặc Custom Metrics (số request/s).
```bash
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10
```

### Vertical Pod Autoscaler (VPA)
Tự động điều chỉnh `requests` và `limits` của Pod dựa trên usage thực tế. Hữu ích cho các stateful service hoặc task khó đoán định resource.
*Lưu ý: Thường VPA yêu cầu restart Pod để apply change.*

## 4. Probes (Health Checks)
Cơ chế để K8s biết tình trạng của ứng dụng.
- **Liveness Probe**: App còn sống không? Nếu chết -> Restart container.
- **Readiness Probe**: App đã sẵn sàng nhận traffic chưa? Nếu chưa -> Rút khỏi Service LoadBalancer.
- **Startup Probe**: Dùng cho app khởi động lâu.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

## 5. Network Policies
Mặc định trong K8s, mọi Pod đều nói chuyện được với nhau (Flat network).
**NetworkPolicy** giúp tạo firewall rule: "Chỉ cho phép Backend nhận traffic từ Frontend, không nhận từ nơi khác".

```yaml
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 5432
```

## 6. Helm Architecture
Dùng Helm để đóng gói ứng dụng (Package Manager).
- **Chart**: Template của resources (Deployment, Service, Ingress...).
- **Values**: File config biến động (environment specific).
- Giúp quản lý versioning và rollback cả một cụm ứng dụng.
