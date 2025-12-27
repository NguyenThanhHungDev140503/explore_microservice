# Kubernetes (K8s) Guide

## Mục Lục
1. [Giới Thiệu Tổng Quan](#giới-thiệu-tổng-quan)
2. [Junior Level – Cơ Bản](#junior-level--cơ-bản)
3. [Middle Level – Trung Cấp](#middle-level--trung-cấp)
4. [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Giới Thiệu Tổng Quan

### Kubernetes là gì?
Kubernetes (K8s) là một nền tảng mã nguồn mở (container orchestration) dùng để tự động hóa việc triển khai, mở rộng (scaling) và quản lý các ứng dụng container hóa.

### Tại sao sử dụng?
- **Tự động hóa**: Giảm thiểu thao tác thủ công khi quản lý hàng trăm container.
- **High Availability**: Đảm bảo ứng dụng luôn chạy, tự phục hồi khi có lỗi.
- **Scalability**: Dễ dàng scale up/down theo nhu cầu traffic.

### Các khái niệm cốt lõi (Architecture)
- **Cluster**: Tập hợp các máy (nodes) chạy K8s.
- **Node**: Máy ảo hoặc vật lý. Có 2 loại:
  - **Control Plane (Master)**: Quản lý cluster (API Server, Scheduler, Controller Manager, etcd).
  - **Worker Node**: Nơi chạy các ứng dụng (Kubelet, Kube-proxy, Container Runtime).
- **Pod**: Đơn vị nhỏ nhất trong K8s, chứa 1 hoặc nhiều container.

---

## Junior Level – Cơ Bản

### 1. Cài đặt môi trường Local
Để bắt đầu, bạn có thể sử dụng các công cụ tạo cluster nhẹ trên máy cá nhân:
- **Minikube**: Tạo single-node cluster chạy trong VM hoặc Docker.
- **Kind (Kubernetes in Docker)**: Chạy cluster K8s sử dụng Docker container làm node.
- **K3s**: Bản phân phối K8s siêu nhẹ.

**Cài đặt kubectl** (Command line tool để giao tiếp với cluster):
```bash
# Ví dụ trên Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### 2. Hello World Example (Pod)
K8s sử dụng file YAML để định nghĩa resources (Declarative).

File: `pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Chạy command:
```bash
kubectl apply -f pod.yaml
kubectl get pods
```

### 3. Deployments
Không nên chạy Pod lẻ (Naked Pod). Hãy dùng **Deployment** để quản lý ReplicaSet (số lượng bản sao) và update ứng dụng.

File: `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3 # Chạy 3 bản sao
  selector:
    matchLabels:
      app: nginx
  template: # Template cho Pod
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
Lợi ích: Tự động restart pod khi chết, hỗ trợ Rolling Update.

### 4. Services
Giúp exposing ứng dụng ra bên ngoài hoặc nội bộ cluster.

File: `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer # Hoặc ClusterIP (default), NodePort
```

---

## Middle Level – Trung Cấp

### 1. Namespaces
Phân chia cluster thành các virtual cluster (ví dụ: `dev`, `staging`, `prod` trên cùng 1 cluster vật lý).

```bash
kubectl create namespace staging
kubectl get pods -n staging
```

### 2. ConfigMaps & Secrets
Tách biệt cấu hình (config) và dữ liệu nhạy cảm khỏi code/image.

- **ConfigMap**: Lưu biến môi trường, file config.
- **Secret**: Lưu password, key (được base64 encoded).

File: `configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://localhost:5432/db"
```

Cách dùng trong Pod:
```yaml
envFrom:
- configMapRef:
    name: app-config
```

### 3. Ingress
Quản lý truy cập từ ngoài vào các Service trong cluster (HTTP/HTTPS routes). Thay thế cho việc dùng quá nhiều LoadBalancer đắt đỏ.
Thường dùng **Nginx Ingress Controller**.

File: `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  rules:
  - host: "my-app.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

### 4. Volumes (Persisting Data)
Container là ephemeral (dữ liệu mất khi container chết). Cần dùng Volumes để lưu dữ liệu lâu dài.
- **PersistentVolume (PV)**: Cung cấp storage thực tế (EBS, NFS, HostPath...).
- **PersistentVolumeClaim (PVC)**: Request storage từ PV.

---

## Tài Liệu Tham Khảo
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [Kubernetes by Example](https://kubernetesbyexample.com/)
