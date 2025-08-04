# Giải thích CronJob ECR Credentials Update

## Tổng quan (Overview)

CronJob này được thiết kế để tự động cập nhật thông tin đăng nhập ECR (Amazon Elastic Container Registry) trong các Kubernetes secrets trên nhiều namespace khác nhau. Điều này đảm bảo rằng các ứng dụng luôn có thể truy cập vào ECR mà không gặp phải lỗi xác thực do token hết hạn.

This CronJob is designed to automatically update ECR (Amazon Elastic Container Registry) credentials in Kubernetes secrets across multiple namespaces. This ensures that applications can always access ECR without encountering authentication errors due to expired tokens.

## Chi tiết cấu hình (Configuration Details)

### 1. Lịch trình thực thi (Schedule)
```yaml
schedule: 0 */10 * * *
```
- **Ý nghĩa**: Chạy vào phút 0 của mỗi 10 phút (0:00, 0:10, 0:20, ...)
- **Tần suất**: 144 lần mỗi ngày
- **Mục đích**: Đảm bảo token ECR luôn được làm mới trước khi hết hạn (ECR tokens thường có thời hạn 12 giờ)

**Meaning**: Runs at minute 0 of every 10 minutes (0:00, 0:10, 0:20, ...)
**Frequency**: 144 times per day
**Purpose**: Ensures ECR tokens are always refreshed before expiration (ECR tokens typically last 12 hours)

### 2. Chính sách đồng thời (Concurrency Policy)
```yaml
concurrencyPolicy: Replace
```
- **Replace**: Nếu job trước đó vẫn đang chạy, nó sẽ bị dừng và thay thế bằng job mới
- **Lợi ích**: Tránh tình trạng nhiều job cùng cập nhật secrets, có thể gây xung đột

**Replace**: If the previous job is still running, it will be stopped and replaced with a new job
**Benefit**: Prevents multiple jobs from updating secrets simultaneously, which could cause conflicts

### 3. Cấu trúc Job Template

#### a) Volumes (Khối lượng lưu trữ)

1. **cache-volume**: 
   ```yaml
   emptyDir:
     sizeLimit: 10Mi
   ```
   - Chia sẻ dữ liệu giữa init container và main container
   - Lưu trữ tạm thời password ECR
   - Giới hạn 10MB để tiết kiệm tài nguyên

2. **aws-envs**: Secret chứa thông tin xác thực AWS
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_DEFAULT_REGION`

3. **ecr-registry**: ConfigMap chứa thông tin registry
   - `DOCKER_SERVER`: URL của ECR registry
   - `DOCKER_USER`: Username (thường là "AWS")
   - `DOCKER_EMAIL`: Email cho Docker registry

4. **ecr-namespace**: ConfigMap chứa danh sách namespace
   - `SECRET_NAMESPACES`: Danh sách các namespace cần cập nhật (phân cách bằng dấu phẩy)
   - `SECRET_NAME`: Tên của secret cần cập nhật

#### b) Init Container: fetch-ecr-credentials

```yaml
image: amazon/aws-cli:latest
```

**Chức năng (Function)**:
1. Sử dụng AWS CLI để lấy password ECR
2. Lưu password vào file `/tmp/password.txt`
3. File này được chia sẻ với main container qua `cache-volume`

**Lệnh thực thi (Command)**:
```bash
aws ecr get-login-password > /tmp/password.txt
```

**Biến môi trường**: Được load từ secret `aws-envs`

#### c) Main Container: kubectl

```yaml
image: bitnami/kubectl:latest
```

**Chức năng chính (Main Functions)**:

1. **Đọc password từ file chia sẻ**:
   ```bash
   DOCKER_PASS=$(cat /tmp/password.txt)
   ```

2. **Xử lý từng namespace**:
   - Kiểm tra namespace có tồn tại không
   - Kiểm tra secret có tồn tại trong namespace không
   - Chỉ cập nhật nếu secret đã tồn tại (không tạo mới)

3. **Cập nhật secret**:
   ```bash
   kubectl create secret docker-registry "$SECRET_NAME" \
       --docker-server="$DOCKER_SERVER" \
       --docker-username="$DOCKER_USER" \
       --docker-password="$DOCKER_PASS" \
       --docker-email="$DOCKER_EMAIL" \
       -n "$NS" \
       --dry-run=client -o yaml | kubectl apply -f -
   ```

### 4. Service Account

```yaml
serviceAccountName: ecr-update-password
```

**Quyền cần thiết (Required Permissions)**:
- `get`, `list` namespaces
- `get`, `list`, `create`, `update` secrets trong các namespace được chỉ định
- Quyền truy cập vào ConfigMaps và Secrets trong `kube-system`

### 5. Cấu hình Job Management

```yaml
ttlSecondsAfterFinished: 60
successfulJobsHistoryLimit: 1
failedJobsHistoryLimit: 1
```

- **TTL**: Job sẽ bị xóa sau 60 giây khi hoàn thành
- **History**: Chỉ giữ lại 1 job thành công và 1 job thất bại gần nhất
- **Mục đích**: Tiết kiệm tài nguyên cluster

## Luồng hoạt động (Workflow)

1. **Khởi tạo (Initialization)**:
   - CronJob tạo Pod theo lịch trình
   - Mount các volumes cần thiết

2. **Init Container**:
   - Sử dụng AWS credentials để xác thực
   - Gọi `aws ecr get-login-password`
   - Lưu password vào `/tmp/password.txt`

3. **Main Container**:
   - Đọc password từ file chia sẻ
   - Lặp qua danh sách namespaces
   - Kiểm tra và cập nhật secrets

4. **Cleanup**:
   - Pod kết thúc
   - Sau 60 giây, Job bị xóa tự động

## Yêu cầu triển khai (Deployment Requirements)

### 1. Prerequisites

1. **AWS Credentials Secret**:
   ```bash
   kubectl create secret generic aws-envs \
     --from-literal=AWS_ACCESS_KEY_ID=your-access-key \
     --from-literal=AWS_SECRET_ACCESS_KEY=your-secret-key \
     --from-literal=AWS_DEFAULT_REGION=your-region \
     -n kube-system
   ```

2. **ECR Registry ConfigMap**:
   ```bash
   kubectl create configmap ecr-registry \
     --from-literal=DOCKER_SERVER=123456789012.dkr.ecr.us-west-2.amazonaws.com \
     --from-literal=DOCKER_USER=AWS \
     --from-literal=DOCKER_EMAIL=your-email@company.com \
     -n kube-system
   ```

3. **ECR Namespace ConfigMap**:
   ```bash
   kubectl create configmap ecr-namespace \
     --from-literal=SECRET_NAMESPACES="default,production,staging" \
     --from-literal=SECRET_NAME=ecr-registry-secret \
     -n kube-system
   ```

4. **Service Account và RBAC**:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: ecr-update-password
     namespace: kube-system
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: ecr-update-password
   rules:
   - apiGroups: [""]
     resources: ["namespaces"]
     verbs: ["get", "list"]
   - apiGroups: [""]
     resources: ["secrets"]
     verbs: ["get", "list", "create", "update", "patch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: ecr-update-password
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: ecr-update-password
   subjects:
   - kind: ServiceAccount
     name: ecr-update-password
     namespace: kube-system
   ```

### 2. Validation

**Kiểm tra CronJob**:
```bash
kubectl get cronjob ecr-creds-update -n kube-system
kubectl describe cronjob ecr-creds-update -n kube-system
```

**Kiểm tra Jobs**:
```bash
kubectl get jobs -n kube-system | grep ecr-creds-update
```

**Kiểm tra Pods**:
```bash
kubectl get pods -n kube-system | grep ecr-creds-update
kubectl logs -f <pod-name> -n kube-system
```

**Kiểm tra Secrets được cập nhật**:
```bash
kubectl get secret ecr-registry-secret -n default -o yaml
```

## Monitoring và Troubleshooting

### 1. Common Issues

1. **AWS Credentials không hợp lệ**:
   - Kiểm tra secret `aws-envs`
   - Đảm bảo IAM user có quyền `ecr:GetAuthorizationToken`

2. **Namespace không tồn tại**:
   - Job sẽ skip namespace này và tiếp tục
   - Kiểm tra logs để xem danh sách namespace được xử lý

3. **Secret không tồn tại**:
   - Job chỉ cập nhật secrets đã có, không tạo mới
   - Cần tạo secret trước khi CronJob chạy

4. **RBAC permissions**:
   - Đảm bảo ServiceAccount có đủ quyền
   - Kiểm tra ClusterRole và ClusterRoleBinding

### 2. Monitoring Commands

```bash
# Xem lịch sử jobs
kubectl get jobs -n kube-system -l job-name=ecr-creds-update

# Xem logs của job gần nhất
kubectl logs -l job-name=ecr-creds-update -n kube-system --tail=100

# Kiểm tra schedule tiếp theo
kubectl get cronjob ecr-creds-update -n kube-system -o yaml | grep -A 5 status
```

## Best Practices

1. **Security**:
   - Sử dụng IAM roles thay vì access keys khi có thể
   - Rotate AWS credentials định kỳ
   - Giới hạn quyền IAM chỉ cho ECR

2. **Monitoring**:
   - Set up alerts cho failed jobs
   - Monitor logs thường xuyên
   - Kiểm tra secret expiration

3. **Resource Management**:
   - Điều chỉnh `ttlSecondsAfterFinished` phù hợp
   - Monitor resource usage của jobs
   - Cân nhắc tần suất chạy dựa trên workload

4. **High Availability**:
   - Đảm bảo multiple nodes có thể chạy job
   - Backup configurations và secrets
   - Test disaster recovery procedures