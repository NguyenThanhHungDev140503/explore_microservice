# Kubernetes (K8s) Documentation

## Giới Thiệu
Bộ tài liệu này cung cấp hướng dẫn toàn diện về **Kubernetes (K8s)**, từ các khái niệm cơ bản cho người mới bắt đầu đến các kiến trúc hệ thống phục vụ doanh nghiệp lớn (Enterprise/Principal level).

## Cấu Trúc Tài Liệu

### 1. [Kubernetes.md](./Kubernetes.md) (Junior/Middle)
- **Dành cho**: Junior, Middle Developers.
- **Nội dung**:
  - Hướng dẫn cài đặt (Minikube, Kind).
  - Architecture cơ bản (Pod, Deployment, Service).
  - Các concept trung cấp (Ingress, ConfigMap, Secrets, Volumes).
  - Cách triển khai ứng dụng Hello World.

### 2. [Advanced-Patterns.md](./Advanced-Patterns.md) (Senior)
- **Dành cho**: Senior Developers, DevOps.
- **Nội dung**:
  - Quản lý tài nguyên chuyên sâu (Requests/Limits, Quotas).
  - Scheduling nâng cao (Affinity, Taints & Tolerations).
  - Chiến lược Autoscaling (HPA, VPA).
  - Network Security & Observability.

### 3. [Principal-Level-Patterns.md](./Principal-Level-Patterns.md) (Principal/Staff)
- **Dành cho**: Principal Engineers, Architects, Tech Leads.
- **Nội dung**:
  - System Design & High Availability Architecture.
  - Multi-tenancy & Isolation.
  - GitOps Workflow (ArgoCD).
  - Service Mesh & Security Hardening.
  - Cost Optimization (FinOps).

### 4. [RESEARCH_SUMMARY.md](./RESEARCH_SUMMARY.md)
- Tổng hợp quá trình nghiên cứu, lý do lựa chọn công nghệ, so sánh (vs Docker Swarm) và định hướng áp dụng cho dự án.

## Cách Sử Dụng Tài Liệu

- **Newbie**: Bắt đầu với `Kubernetes.md` để hiểu "Pod là gì" và chạy được app đầu tiên.
- **DevOps/Senior**: Tham khảo `Advanced-Patterns.md` để tối ưu performance cluster và config security.
- **Architect**: Đọc `Principal-Level-Patterns.md` để thiết kế hệ thống chịu tải lớn, an toàn và dễ vận hành.

## Quick Start

Yêu cầu: Đã cài Docker và Minikube.

```bash
# 1. Start cluster
minikube start

# 2. Tạo Deployment
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4

# 3. Expose Service
kubectl expose deployment hello-minikube --type=NodePort --port=8080

# 4. Check URL
minikube service hello-minikube
```

## External Resources
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [CNCF Landscape](https://landscape.cncf.io/)
