# Spring Boot Hello World + Argo CD

Dự án nhỏ này minh họa chuỗi triển khai GitOps:

```text
push/PR -> GitHub Actions: test -> build image -> ghi image SHA vào Git
                                                |
                                                v
                               Argo CD theo dõi Git -> sync Kustomize -> Kubernetes
```

`Argo CD` không build image. Nó là phần CD: liên tục so sánh trạng thái Kubernetes với Git và đưa cluster về đúng trạng thái trong Git. CI ở đây là GitHub Actions; CI chỉ cập nhật **tag bất biến** (Git SHA) trong `k8s/overlays/dev/kustomization.yaml`. Đây là điểm GitOps quan trọng nhất để quan sát khi học.

## Cấu trúc

- `src/`: API `GET /` trả về lời chào; Actuator cung cấp health probes.
- `Dockerfile`: đóng gói ứng dụng thành container không chạy bằng root.
- `k8s/base/`: Deployment và Service dùng chung.
- `k8s/overlays/dev/`: môi trường dev và tag image được CI thay đổi.
- `argocd/application-dev.yaml`: đối tượng Argo CD Application, theo dõi overlay dev.
- `.github/workflows/ci-cd.yaml`: CI kiểm tra PR; với `main`, publish image lên GHCR và commit tag SHA vào manifest.

## 1. Chạy ứng dụng local

Yêu cầu: Java 21 và Maven.

```bash
mvn verify
mvn spring-boot:run
curl http://localhost:8080/
curl http://localhost:8080/actuator/health
```

Kết quả endpoint đầu tiên là JSON chứa `Hello from Spring Boot + Argo CD!`.

## 2. Tạo Kubernetes local và cài Argo CD

Ví dụ với [kind](https://kind.sigs.k8s.io/):

```bash
kind create cluster --name argocd-lab
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=180s
```

Cài [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) theo hệ điều hành của bạn. Mở giao diện và lấy mật khẩu admin ban đầu:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
argocd admin initial-password -n argocd
```

Sau đó truy cập `https://localhost:8081`, đăng nhập `admin` với mật khẩu trên. Chấp nhận cảnh báo chứng chỉ tự ký chỉ trong môi trường lab.

## 3. Kết nối Git repository với Argo CD

1. Tạo Git repository mới trên GitHub và đẩy **toàn bộ** thư mục này lên nhánh `main`.
2. Sửa `repoURL` trong `argocd/application-dev.yaml` thành URL HTTPS thật của repository.
3. Áp dụng Application:

   ```bash
   kubectl apply -f argocd/application-dev.yaml
   argocd app get hello-argocd-dev
   ```

4. Nếu repository/image GHCR là private, cấp quyền đọc image cho cluster (hoặc chuyển package sang public khi học). Với Git private, thêm repository credentials trong Argo CD trước khi tạo Application.

Argo CD sẽ tạo namespace `hello-dev`, áp dụng Kustomize, và tự sync khi Git thay đổi. Kiểm tra:

```bash
kubectl get pods -n hello-dev
kubectl port-forward svc/hello-argocd -n hello-dev 8080:80
curl http://localhost:8080/
```

Lần sync đầu sẽ dùng placeholder `ghcr.io/replace-me/...`, nên Deployment chỉ chạy được sau lần push `main` đầu tiên khiến workflow publish image và cập nhật `newName`/`newTag`. Nếu muốn chạy trước khi có CI, tự build/load image vào kind rồi thay hai giá trị này.

## 4. Bài thực hành nên làm

1. Đổi câu chào, push `main`, xem commit GitOps do bot tạo và xem Argo CD chuyển `OutOfSync` rồi `Synced`.
2. Xóa Service bằng `kubectl delete service hello-argocd -n hello-dev`; bật `selfHeal` sẽ để Argo CD tạo lại nó.
3. Xóa `service.yaml` khỏi `k8s/base/kustomization.yaml` và push; `prune: true` sẽ xóa Service khỏi cluster.
4. Tắt `automated` rồi sync thủ công từ UI/CLI để phân biệt deploy tự động và thủ công.

## Ghi chú production

Đây là lab một repository để dễ học. Trong công ty, thường tách application repo và environment/GitOps repo, dùng image digest thay vì tag, quản lý secret qua External Secrets/Sealed Secrets, giới hạn Argo CD Project/RBAC, và cấu hình ingress/TLS/observability theo chuẩn nội bộ.
