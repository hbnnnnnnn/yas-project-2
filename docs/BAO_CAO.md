# Báo Cáo Đồ Án 2: Xây Dựng Hệ Thống CI/CD và Service Mesh
# Dự án: YAS — Yet Another Shop

---

## Mục Lục

1. [Tổng Quan Dự Án](#1-tổng-quan-dự-án)
2. [Kubernetes Cluster](#2-kubernetes-cluster)
3. [CI Pipeline — Tự Động Build Image](#3-ci-pipeline--tự-động-build-image)
4. [Job developer_build — Deploy Theo Branch](#4-job-developer_build--deploy-theo-branch)
5. [Job Cleanup](#5-job-cleanup)
6. [CD Môi Trường Dev — Auto Deploy Khi Push main](#6-cd-môi-trường-dev--auto-deploy-khi-push-main)
7. [CD Môi Trường Staging — Deploy Theo Tag Release](#7-cd-môi-trường-staging--deploy-theo-tag-release)
8. [Nâng Cao: ArgoCD GitOps](#8-nâng-cao-argocd-gitops)
9. [Nâng Cao: Service Mesh với Istio](#9-nâng-cao-service-mesh-với-istio)
10. [Các Vấn Đề Gặp Phải và Cách Giải Quyết](#10-các-vấn-đề-gặp-phải-và-cách-giải-quyết)

---

## 1. Tổng Quan Dự Án

### 1.1 Mô Tả

Dự án xây dựng toàn bộ hạ tầng CI/CD và Service Mesh cho ứng dụng microservice **YAS (Yet Another Shop)** — một ứng dụng thương mại điện tử gồm 13 microservice viết bằng Java Spring Boot.

Nhiệm vụ **không** phải viết code ứng dụng mà là xây dựng toàn bộ hệ thống triển khai xung quanh nó:
- Tự động hoá quá trình build, test, đóng gói Docker image (CI)
- Tự động hoá quá trình triển khai lên Kubernetes (CD)
- Cấu hình Service Mesh để mã hoá traffic và kiểm soát kết nối giữa các service

### 1.2 Kiến Trúc Tổng Thể

```
Developer push code
        │
        ▼
   GitHub (yas-project-2)
        │
        ▼ webhook
   Jenkins CI
   ├── Phát hiện service nào thay đổi
   ├── Build Docker image → tag = commit ID
   ├── Push lên Docker Hub (hbnhbn/<service>:<commitid>)
   └── Cập nhật file values trong repo yas-gitops
        │
        ▼
   GitHub (yas-gitops)
   dev/<service>.values.yaml   ← ArgoCD theo dõi
        │
        ▼
   ArgoCD ApplicationSet
   ├── Tạo tự động 1 Application cho mỗi service
   ├── Multi-source: chart từ yas-project-2 + values từ yas-gitops
   └── Auto-sync, auto-prune
        │
        ▼
   Kubernetes (Minikube)
   ├── namespace: dev      ← 13 services (pods 2/2 với Istio sidecar)
   ├── namespace: staging  ← 13 services (release tags)
   ├── namespace: postgres / kafka / keycloak / elasticsearch / redis
   └── namespace: istio-system (istiod, Kiali, Prometheus)
```

### 1.3 Danh Sách File Quan Trọng

| File | Mục Đích |
|---|---|
| `Jenkinsfile` | CI pipeline: phát hiện service thay đổi, build & push image |
| `Jenkinsfile.developer_build` | Developer chọn branch riêng cho từng service để test |
| `Jenkinsfile.deploy_dev` | CD pipeline: push lên main → deploy vào namespace dev |
| `Jenkinsfile.deploy_staging` | CD pipeline: push tag v1.2.3 → deploy vào namespace staging |
| `Jenkinsfile.cleanup_dev` | Reset toàn bộ image tag dev về `main` |
| `k8s/argocd/applicationset-dev.yaml` | ArgoCD ApplicationSet cho môi trường dev |
| `k8s/argocd/applicationset-staging.yaml` | ArgoCD ApplicationSet cho môi trường staging |
| `k8s/istio/peer-authentication.yaml` | Bật mTLS STRICT cho namespace dev |
| `k8s/istio/authz-deny-all.yaml` | Chặn toàn bộ traffic mặc định |
| `k8s/istio/authz-allow-storefront-bff.yaml` | Chỉ cho phép storefront-bff gọi các service |
| `k8s/istio/authz-allow-backoffice-bff.yaml` | Chỉ cho phép backoffice-bff gọi các service |
| `k8s/istio/virtualservice-tax.yaml` | Cấu hình retry policy cho service tax |

### 1.4 Danh Sách 13 Service Triển Khai

| Service | Loại | Mô Tả |
|---|---|---|
| `tax` | Backend | Tính thuế |
| `product` | Backend | Quản lý sản phẩm |
| `cart` | Backend | Giỏ hàng |
| `order` | Backend | Đơn hàng |
| `customer` | Backend | Khách hàng |
| `inventory` | Backend | Tồn kho |
| `media` | Backend | Upload media |
| `search` | Backend | Tìm kiếm (Elasticsearch) |
| `storefront-bff` | BFF Gateway | API gateway cho storefront |
| `backoffice-bff` | BFF Gateway | API gateway cho backoffice |
| `storefront-ui` | Frontend | Giao diện cửa hàng (Next.js) |
| `backoffice-ui` | Frontend | Giao diện quản trị (Next.js) |
| `swagger-ui` | Docs | Swagger API documentation |

---

## 2. Kubernetes Cluster

### 2.1 Mô Tả

Cluster Kubernetes được triển khai bằng **Minikube** chạy trên máy local. Sử dụng nhiều namespace để cô lập từng thành phần:

| Namespace | Nội dung |
|---|---|
| `dev` | 13 service ứng dụng (môi trường phát triển) |
| `staging` | 13 service ứng dụng (môi trường kiểm thử) |
| `postgres` | Cơ sở dữ liệu PostgreSQL (Zalando Operator) |
| `kafka` | Message broker Kafka (Strimzi KRaft) |
| `keycloak` | Xác thực OAuth2/JWT |
| `elasticsearch` | Search engine (ECK) |
| `redis` | Cache / session |
| `argocd` | GitOps controller |
| `istio-system` | Service mesh (istiod, Kiali, Prometheus) |

**Lý do tách namespace:** Mỗi namespace hoạt động độc lập. Sự cố ở namespace `dev` không ảnh hưởng đến cơ sở dữ liệu ở namespace `postgres`. Các service kết nối cross-namespace qua DNS đầy đủ, ví dụ: `postgresql.postgres.svc.cluster.local`.

### 2.2 Lệnh Chụp Màn Hình

```bash
# [Ảnh 1] Thông tin cluster và node
kubectl cluster-info
kubectl get nodes -o wide

# [Ảnh 2] Danh sách tất cả namespace
kubectl get namespaces

# [Ảnh 3] Các pod hạ tầng trong namespace riêng
kubectl get pods -n postgres
kubectl get pods -n kafka
kubectl get pods -n keycloak
kubectl get pods -n elasticsearch
kubectl get pods -n redis
```

---

## 3. CI Pipeline — Tự Động Build Image

### 3.1 Mô Tả

Mỗi khi developer push code lên GitHub, Jenkins tự động phát hiện service nào thay đổi, build Docker image và đẩy lên Docker Hub với tag là **commit ID** của branch đó.

**Luồng hoạt động:**
1. Developer push code → GitHub gửi webhook đến Jenkins
2. Jenkins so sánh commit hiện tại với commit trước để xác định file nào thay đổi
3. Với mỗi service thay đổi: chạy `docker build` và `docker push`
4. Tag image: `hbnhbn/<service>:<commitid>` (ví dụ: `hbnhbn/tax:a3f92b1`)
5. Jenkins cập nhật file `dev/<service>.values.yaml` trong repo `yas-gitops` với tag mới

### 3.2 Cấu Trúc Jenkinsfile

```groovy
// Ví dụ logic phát hiện service thay đổi
def changedServices = []
def allServices = ['tax', 'product', 'cart', 'order', ...]

for (service in allServices) {
    def changed = sh(script: "git diff --name-only HEAD~1 | grep ^${service}/", returnStatus: true)
    if (changed == 0) changedServices.add(service)
}

// Build và push chỉ các service thay đổi
for (service in changedServices) {
    sh "docker build -t hbnhbn/${service}:${commitId} ./${service}/"
    sh "docker push hbnhbn/${service}:${commitId}"
}
```

### 3.3 Lệnh Chụp Màn Hình

```bash
# [Ảnh 4] Xem các stage trong Jenkinsfile
cat Jenkinsfile | grep -A3 'stage('
```

**Trong Jenkins UI:**
- **[Ảnh 5]** Mở pipeline chính → chọn một build gần đây → nhấn **Console Output**
- Cuộn để thấy: service nào được phát hiện thay đổi, lệnh `docker build`, lệnh `docker push`, tag = commit ID

**Trên Docker Hub:**
- **[Ảnh 6]** Mở `https://hub.docker.com/r/hbnhbn/tax/tags`
- Thấy các tag dạng commit ID như `a3f92b1c`, `285fac57`, ...

---

## 4. Job developer_build — Deploy Theo Branch

### 4.1 Mô Tả

Job `developer_build` cho phép developer chọn **branch riêng cho từng service** để test. Ví dụ: developer đang làm việc ở branch `dev_tax_service` và muốn xem kết quả trên cluster:

- Vào job `developer_build`
- Điền parameter: `TAX_BRANCH = dev_tax_service`
- Các service còn lại để `main`
- Jenkins sẽ build image `hbnhbn/tax:<commit-id-của-dev_tax_service>` và deploy service tax với image đó, các service khác giữ nguyên tag `main`

### 4.2 Lệnh Chụp Màn Hình

```bash
# [Ảnh 7] Xem cấu trúc job
cat Jenkinsfile.developer_build | head -80
```

**Trong Jenkins UI:**
- **[Ảnh 8]** Mở job `developer_build` → nhấn **Build with Parameters** → thấy form nhập branch cho từng service (mặc định `main`)
- **[Ảnh 9]** Nhập `dev_tax_service` vào ô TAX_BRANCH → Build → Console Output

```bash
# [Ảnh 10] Sau khi job chạy xong — kiểm tra image của tax
kubectl get deployment tax -n dev -o jsonpath='{.spec.template.spec.containers[0].image}'
# Kết quả: hbnhbn/tax:<commit-id-của-dev_tax_service>

# Các service khác vẫn dùng tag main
kubectl get pods -n dev -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image' | grep -v istio
```

---

## 5. Job Cleanup

### 5.1 Mô Tả

Sau khi developer test xong, job `cleanup_dev` reset toàn bộ image tag trong namespace `dev` về `main`, xoá các deployment riêng của developer.

### 5.2 Lệnh Chụp Màn Hình

```bash
# [Ảnh 11] Xem Jenkinsfile.cleanup_dev
cat Jenkinsfile.cleanup_dev
```

**Trong Jenkins UI:**
- **[Ảnh 12]** Mở job `cleanup_dev` → Build → Console Output (thấy `yq` reset từng service về `main`)

---

## 6. CD Môi Trường Dev — Auto Deploy Khi Push main

### 6.1 Mô Tả

Mỗi khi có commit vào nhánh `main`, pipeline `deploy_dev` tự động chạy:
1. Lấy commit ID mới nhất
2. Cập nhật file `dev/<service>.values.yaml` trong repo `yas-gitops` với tag mới
3. ArgoCD phát hiện thay đổi trong repo `yas-gitops` → tự động sync → deploy phiên bản mới vào namespace `dev`

**Cấu hình GitOps (hai repo):**
- `yas-project-2`: chứa Helm charts (cái *gì* để deploy)
- `yas-gitops`: chứa values files (cấu hình *thế nào* cho từng môi trường)

### 6.2 Lệnh Chụp Màn Hình

```bash
# [Ảnh 13] Xem Jenkinsfile.deploy_dev
cat Jenkinsfile.deploy_dev | head -50

# [Ảnh 14] Xem file values trong gitops repo — thấy tag = commit ID
cat /Users/hbn/Documents/yas-gitops/dev/tax.values.yaml
# backend.image.tag = <commitid>

# [Ảnh 15] Tất cả pod dev đang chạy 2/2
kubectl get pods -n dev
```

**Trong ArgoCD UI (https://localhost:8080):**
```bash
# Lấy mật khẩu ArgoCD
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
- **[Ảnh 16]** Danh sách 13 app dev — tất cả `Synced` + `Healthy`
- **[Ảnh 17]** Click vào app `yas-dev-tax` → xem phần **Sources** (2 repo: chart + values) và image tag hiện tại

---

## 7. CD Môi Trường Staging — Deploy Theo Tag Release

### 7.1 Mô Tả

Staging dùng cho release candidate. Khi team lead push git tag dạng `v1.2.3`:
1. Jenkins phát hiện tag → chạy pipeline `deploy_staging`
2. Build lại image với tag là version release (ví dụ `hbnhbn/tax:v1.2.3`)
3. Cập nhật `staging/<service>.values.yaml` trong `yas-gitops`
4. ArgoCD sync → deploy vào namespace `staging`

**Sự khác biệt dev vs staging:**
- `dev`: tag = commit ID, cập nhật liên tục mỗi push
- `staging`: tag = version release (`v1.2.3`), chỉ cập nhật khi có release chính thức

### 7.2 Lệnh Chụp Màn Hình

```bash
# [Ảnh 18] Xem Jenkinsfile.deploy_staging
cat Jenkinsfile.deploy_staging | head -60

# [Ảnh 19] So sánh tag dev vs staging
echo "=== DEV ===" && cat /Users/hbn/Documents/yas-gitops/dev/tax.values.yaml
echo "=== STAGING ===" && cat /Users/hbn/Documents/yas-gitops/staging/tax.values.yaml

# [Ảnh 20] Các pod staging
kubectl get pods -n staging
```

**Trong ArgoCD UI:**
- **[Ảnh 21]** Lọc theo `staging` — thấy 13 app staging `Synced` + `Healthy` (chạy song song với 13 app dev)

---

## 8. Nâng Cao: ArgoCD GitOps

### 8.1 Mô Tả

Thay vì tạo thủ công 26 ArgoCD Application (13 dev + 13 staging), chúng ta dùng **ApplicationSet** — một resource tự động sinh ra Application cho từng service dựa trên danh sách cấu hình.

**Ưu điểm:**
- Chỉ cần viết 1 template YAML → tự tạo 13 Application
- `prune: true`: khi xoá service khỏi danh sách → ArgoCD tự xoá pod và resource trên cluster
- `selfHeal: true`: nếu ai sửa trực tiếp trên cluster → ArgoCD tự revert về trạng thái trong Git
- **Multi-source**: mỗi Application kéo chart từ `yas-project-2` VÀ values từ `yas-gitops` cùng lúc

### 8.2 Cấu Trúc ApplicationSet

```yaml
# k8s/argocd/applicationset-dev.yaml (đơn giản hoá)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
    - list:
        elements:
          - service: tax
          - service: product
          - service: cart
          # ... 13 service
  template:
    spec:
      sources:
        - repoURL: https://github.com/...yas-project-2   # Helm chart
          path: k8s/charts/{{ "{{" }}service{{ "}}" }}
        - repoURL: https://github.com/...yas-gitops       # Values file
          path: dev
          helm:
            valueFiles:
              - $values/dev/{{ "{{" }}service{{ "}}" }}.values.yaml
      syncPolicy:
        automated:
          prune: true       # Tự xoá resource thừa
          selfHeal: true    # Tự revert nếu bị sửa tay
```

### 8.3 Lệnh Chụp Màn Hình

```bash
# [Ảnh 22] Xem ApplicationSet dev
cat k8s/argocd/applicationset-dev.yaml

# [Ảnh 23] Xem ApplicationSet staging
cat k8s/argocd/applicationset-staging.yaml
```

**Trong ArgoCD UI:**
- **[Ảnh 24]** Toàn bộ 26 Application (13 dev + 13 staging) trong một màn hình

---

## 9. Nâng Cao: Service Mesh với Istio

### 9.1 Kiến Trúc Service Mesh

Istio inject một **Envoy proxy** (sidecar) vào mỗi pod trong namespace `dev`. Toàn bộ traffic giữa các service đi qua proxy này, cho phép:

| Tính năng | Cơ chế |
|---|---|
| mTLS | PeerAuthentication STRICT |
| Kiểm soát kết nối | AuthorizationPolicy |
| Retry tự động | VirtualService |
| Quan sát topology | Kiali |

**Tại sao pod hiển thị `2/2`?**
Container 1 = ứng dụng, Container 2 = Envoy sidecar proxy (do Istio tự động inject vì namespace có label `istio-injection=enabled`).

---

### 9.2 mTLS — Mã Hoá Toàn Bộ Traffic

**Mô tả:** Mọi kết nối giữa các service trong namespace `dev` đều phải dùng mTLS (Mutual TLS). Cả hai phía đều phải xuất trình chứng chỉ X.509 hợp lệ (do Istio CA cấp). Pod không có sidecar (không thuộc mesh) sẽ bị từ chối kết nối ngay lập tức.

**File cấu hình:** `k8s/istio/peer-authentication.yaml`

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: dev
spec:
  mtls:
    mode: STRICT   # Bắt buộc mTLS, từ chối plain HTTP
```

**Lệnh chụp màn hình:**

```bash
# [Ảnh 25] Xem policy mTLS
kubectl get peerauthentication -n dev
kubectl describe peerauthentication default -n dev
# → Thấy: mtls.mode = STRICT

# [Ảnh 26] Namespace dev có label Istio injection
kubectl get namespace dev --show-labels
# → Thấy: istio-injection=enabled

# [Ảnh 27] Tất cả pod 2/2 (app + sidecar)
kubectl get pods -n dev
# → Cột READY = 2/2 cho tất cả pod
```

**Test mTLS — kết nối từ ngoài mesh bị từ chối:**

```bash
# Tạo pod test trong namespace default (không có sidecar)
kubectl run test-pod --image=curlimages/curl -n default --restart=Never -- sleep 3600

# Chờ pod ready
kubectl get pod test-pod -n default

# [Ảnh 28] Test kết nối từ ngoài mesh → bị từ chối
kubectl exec -n default test-pod -- \
  curl -v http://tax.dev.svc.cluster.local/tax/api/taxes --max-time 5
# Kết quả mong đợi: Connection refused
# Giải thích: Envoy proxy trên pod tax từ chối vì không có TLS handshake
```

---

### 9.3 AuthorizationPolicy — Kiểm Soát Kết Nối

**Mô tả:** Chặn toàn bộ traffic mặc định (`deny-all`), sau đó chỉ cho phép `storefront-bff` và `backoffice-bff` gọi các service khác. Ngay cả khi pod ở trong mesh với mTLS hợp lệ, nếu không được cấp phép thì vẫn bị chặn.

**Các file cấu hình:**

```yaml
# k8s/istio/authz-deny-all.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: dev
spec: {}   # Không có rule = từ chối tất cả
```

```yaml
# k8s/istio/authz-allow-storefront-bff.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-storefront-bff
  namespace: dev
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/dev/sa/storefront-bff"
```

**Lệnh chụp màn hình:**

```bash
# [Ảnh 29] Xem tất cả AuthorizationPolicy
kubectl get authorizationpolicy -n dev
# → Thấy: deny-all, allow-storefront-bff, allow-backoffice-bff

# Xem file YAML
cat k8s/istio/authz-deny-all.yaml
cat k8s/istio/authz-allow-storefront-bff.yaml

# [Ảnh 30] Test pod KHÔNG được cấp phép → 503
kubectl exec -n dev curl-test -- \
  curl -v http://tax.dev.svc.cluster.local/tax/api/taxes --max-time 5
# Kết quả mong đợi: 503 Service Unavailable
# Giải thích: curl-test dùng default service account → bị deny-all chặn tại proxy

# [Ảnh 31] Test pod ĐƯỢC cấp phép → 401 (qua được proxy, bị Spring Security chặn)
kubectl exec -n dev curl-authorized -- \
  curl -v http://tax.dev.svc.cluster.local/tax/api/taxes --max-time 5
# Kết quả mong đợi: 401 Unauthorized
# Giải thích: curl-authorized dùng storefront-bff service account
#             → vượt qua được proxy (AuthorizationPolicy cho phép)
#             → bị Spring Security chặn vì thiếu JWT token
```

**Bảng so sánh kết quả:**

| Pod | Service Account | Kết Quả | Giải Thích |
|---|---|---|---|
| `curl-test` | `default` | `503` | Bị chặn tại Envoy proxy (deny-all) |
| `curl-authorized` | `storefront-bff` | `401` | Qua được proxy, bị app chặn (thiếu JWT) |
| `test-pod` (default ns) | - | Connection refused | Không có sidecar, không vào được mesh |

---

### 9.4 Retry Policy — Tự Động Retry Khi Lỗi 5xx

**Mô tả:** Khi service `tax` trả về lỗi 5xx hoặc tạm thời không khả dụng, Envoy proxy tự động retry đến 3 lần mà không cần service gọi (order, cart...) phải tự xử lý logic này.

**File cấu hình:** `k8s/istio/virtualservice-tax.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tax
  namespace: dev
spec:
  hosts:
  - tax
  http:
  - retries:
      attempts: 3           # Retry tối đa 3 lần
      perTryTimeout: 2s     # Mỗi lần retry timeout sau 2 giây
      retryOn: 5xx,reset,connect-failure
    route:
    - destination:
        host: tax
```

**Lệnh chụp màn hình:**

```bash
# [Ảnh 32] Xem VirtualService
kubectl get virtualservice tax -n dev -o yaml

# [Ảnh 33] Bằng chứng retry — scale tax về 0 để trigger retry
kubectl scale deployment tax -n dev --replicas=0

# Thực hiện 1 request từ pod được cấp phép
kubectl exec -n dev curl-authorized -- \
  curl -v http://tax.dev.svc.cluster.local/tax/api/taxes --max-time 30

# Xem thống kê retry trong Envoy proxy
kubectl exec -n dev curl-authorized -c istio-proxy -- \
  pilot-agent request GET stats | grep upstream_rq_retry

# Scale lại
kubectl scale deployment tax -n dev --replicas=1
```

**Giải thích kết quả:** Mặc dù chỉ thực hiện 1 request, Envoy proxy đã gửi 4 request đến upstream (1 lần gốc + 3 lần retry). Counter `upstream_rq_retry` tăng lên, xác nhận retry đã hoạt động.

---

### 9.5 Kiali — Topology Service Mesh

**Mô tả:** Kiali cung cấp giao diện đồ hoạ cho toàn bộ service mesh — hiển thị service nào đang giao tiếp với service nào, tỉ lệ lỗi, và trạng thái mTLS trên từng kết nối.

```bash
# Mở Kiali dashboard
istioctl dashboard kiali &
# Truy cập: http://localhost:20001
```

**Hướng dẫn chụp màn hình trong Kiali:**

1. Vào **Graph** → chọn namespace `dev` → thời gian `Last 10m`
2. **[Ảnh 34]** Chụp toàn bộ topology graph
   - Các nút (node) = các service
   - Các mũi tên = traffic đang chạy
   - **Biểu tượng khoá trên mũi tên** = kết nối có mTLS
3. **[Ảnh 35]** Click vào node `tax` → xem metrics (số request, status code)
4. **[Ảnh 36]** Chú thích flow:
   - Browser → storefront-ui → storefront-bff (có khoá = mTLS)
   - storefront-bff → tax / product / cart / ... (có khoá = mTLS)
   - Không có mũi tên giữa các service backend (AuthorizationPolicy chặn)

---

## 10. Các Vấn Đề Gặp Phải và Cách Giải Quyết

| # | Vấn Đề | Nguyên Nhân | Giải Quyết |
|---|---|---|---|
| 1 | Strimzi CRD không tìm thấy | Cache Helm API discovery cũ | `rm -rf ~/.kube/cache` + re-apply |
| 2 | Strimzi API version mismatch | Strimzi 1.0.0 đổi từ `v1beta2` → `v1` | Cập nhật tất cả template |
| 3 | ZooKeeper bị xoá | Strimzi 1.0.0 chỉ hỗ trợ KRaft | Viết lại Kafka cluster sang KRaft mode |
| 4 | Kafka version không hỗ trợ | Strimzi 1.0.0 chỉ hỗ trợ Kafka 4.x | Đổi từ 3.9.0 → 4.1.0 |
| 5 | ApplicationSet CRD bị thiếu | Race condition khi cài ArgoCD | Re-apply với `--server-side` |
| 6 | Không có pod nào được deploy | GitOps repo chỉ có values file | Chuyển sang multi-source ApplicationSet |
| 7 | Helm dependency không đóng gói | `file://` path không resolve được từ ArgoCD | Pre-package tất cả `backend-0.1.0.tgz` |
| 8 | ServiceMonitor CRD thiếu | Prometheus Operator chưa cài | Tắt `serviceMonitor.enabled` trong ApplicationSet |
| 9 | ConfigMap không có trong namespace dev | ConfigMap chỉ tồn tại ở namespace `yas` | Thêm `yas-configuration` vào ApplicationSet |
| 10 | UI probe timeout | Không có `initialDelaySeconds` + Istio overhead | Thêm `initialDelaySeconds: 30`, `failureThreshold: 12` |
| 11 | DestinationRule xung đột mTLS | `ISTIO_MUTUAL` mâu thuẫn với auto-mTLS từ Istio 1.5+ | Xoá DestinationRule, dùng auto-mTLS |
| 12 | NodePort không accessible | Docker driver không expose NodePort | Dùng `kubectl port-forward` |
| 13 | Tất cả service backend OOMKilled | JVM heap không giới hạn trên node Minikube | Thêm `JAVA_TOOL_OPTIONS=-Xmx512m` + limit 700Mi |
| 14 | Kafka broker OOMKilled | JVM heap không giới hạn | Thêm `jvmOptions: -Xmx512m` trong KafkaNodePool |
| 15 | search service CrashLoopBackOff | Bug trong `co.elastic.clients`: gửi HEAD request rồi cố đọc body (HTTP không cho phép body trong HEAD response) | Build lại image với `@Document(createIndex=false)`, tạo index `product` thủ công trong ES |

---

## Phụ Lục — Trạng Thái Hệ Thống Hiện Tại

| Thành Phần | Trạng Thái |
|---|---|
| Jenkins CI pipelines | ✅ Hoàn thành |
| Kubernetes cluster (Minikube) | ✅ Đang chạy |
| PostgreSQL, Redis, Keycloak, Kafka, Elasticsearch | ✅ Đang chạy |
| ArgoCD ApplicationSet (dev) — 13 service | ✅ Synced |
| ArgoCD ApplicationSet (staging) — 13 service | ✅ Synced |
| Istio (istiod + Kiali + Prometheus) | ✅ Đang chạy |
| Istio sidecar injection trong namespace dev | ✅ Hoạt động (tất cả pod 2/2) |
| mTLS STRICT | ✅ Áp dụng |
| AuthorizationPolicy (deny-all + allow BFFs) | ✅ Áp dụng |
| VirtualService retry cho tax | ✅ Áp dụng |
| Tất cả 13 service | ✅ Đang chạy |
