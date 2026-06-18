# Ký image (Cosign key-based) + bật verify ở admission

## 1. Sinh cặp khoá
```bash
cosign generate-key-pair          # tạo cosign.key (private) + cosign.pub (public)
```
- `cosign.key` (private): **KHÔNG commit**. Đưa vào GitHub Secrets:
  - `COSIGN_PRIVATE_KEY` = nội dung `cosign.key`
  - `COSIGN_PASSWORD`    = mật khẩu khi generate
- `cosign.pub` (public): dán nội dung vào **2 chỗ**:
  - `signing/cosign.pub`
  - `policies/cluster-image-policy.yaml` → `authorities[0].key.data`

## 2. Ký image
- **CI (khi bật được Actions)**: `.github/workflows/build-push.yml` tự Trivy scan +
  Cosign sign theo digest sau khi build.
- **Local (fork tắt Actions)**:
  ```bash
  docker build -t ghcr.io/taitri2004/w10-api:0.0.1 src/api
  trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed ghcr.io/taitri2004/w10-api:0.0.1
  docker push ghcr.io/taitri2004/w10-api:0.0.1
  cosign sign --key cosign.key ghcr.io/taitri2004/w10-api:0.0.1
  # minikube không cần registry: minikube image load ghcr.io/taitri2004/w10-api:0.0.1 -p w10
  ```

## 3. Bật enforce (ĐÚNG THỨ TỰ — bẫy)
Gắn label cho namespace **SAU** khi image đã ký (gắn trước sẽ chặn luôn app api):
```bash
kubectl label ns demo     policy.sigstore.dev/include=true
kubectl label ns payments policy.sigstore.dev/include=true
```

## 4. Kiểm chứng
```bash
# image CHƯA ký -> reject
kubectl -n demo run bad --image=ghcr.io/taitri2004/w10-api:unsigned --dry-run=server
# image ĐÃ ký  -> pass (app api/payments-api chạy xanh)
```
