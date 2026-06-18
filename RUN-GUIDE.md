# Hướng dẫn chạy & nộp bài W10 (làm trên PowerShell, context `w10`)

> Thứ tự làm từ trên xuống. Mỗi phần có **Checkpoint** để biết đã đúng chưa.
> 4 chứng minh của BÀI LỚN (Phần 5) là phần chấm chính — ưu tiên làm xong trước.

---

## Phần 0 — Cài tool (1 lần)
```powershell
# Mở PowerShell ADMIN cho choco
choco install cosign -y
choco install trivy -y      # tuỳ chọn (scan local)
cosign version
kubectl version --client
minikube version
```
**Checkpoint:** `cosign version` in ra version.

---

## Phần 1 — Sinh khoá Cosign + nhúng public key
```powershell
cd C:\Users\ADMIN\Downloads\W10-exercise
cosign generate-key-pair        # nhập password (NHỚ password) -> tạo cosign.key + cosign.pub
```
Nhúng public key vào 2 file (`signing/cosign.pub` và `policies/cluster-image-policy.yaml`):
```powershell
$pub = (Get-Content cosign.pub -Raw).TrimEnd()
# 1) signing/cosign.pub
Set-Content -Path signing/cosign.pub -Value $pub -NoNewline
# 2) policies/cluster-image-policy.yaml — thay block placeholder (thụt 10 space)
$indented = ($pub -split "`n" | ForEach-Object { "          $_" }) -join "`n"
$p = Get-Content policies/cluster-image-policy.yaml -Raw
$p = [regex]::Replace($p, "(?s)          -----BEGIN PUBLIC KEY-----.*?-----END PUBLIC KEY-----", $indented)
Set-Content -Path policies/cluster-image-policy.yaml -Value $p -NoNewline
```
**Checkpoint:** `policies/cluster-image-policy.yaml` không còn chữ `REPLACE_WITH_COSIGN_PUBLIC_KEY`.
> `cosign.key` (private) đã được `.gitignore` — KHÔNG bị commit. Yên tâm.

---

## Phần 2 — Build image API
**Cách A (đơn giản, đủ cho Lab1 + bài lớn, KHÔNG cần registry):**
```powershell
docker build -t ghcr.io/taitri2004/w10-api:0.0.1 src/api
minikube image load ghcr.io/taitri2004/w10-api:0.0.1 -p w10
```
**Cách B (đầy đủ — cần cho chứng minh "image đã ký pass / chưa ký reject"):**
```powershell
# Cần GitHub PAT có quyền write:packages
$env:GHCR_TOKEN | docker login ghcr.io -u taitri2004 --password-stdin
docker build -t ghcr.io/taitri2004/w10-api:0.0.1 src/api
trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed ghcr.io/taitri2004/w10-api:0.0.1
docker push ghcr.io/taitri2004/w10-api:0.0.1
cosign sign --key cosign.key ghcr.io/taitri2004/w10-api:0.0.1     # nhập password
```
**Checkpoint:** image tồn tại (Cách A: `minikube image ls -p w10 | findstr w10-api`).

---

## Phần 3 — Push fork lên GitHub (ArgoCD pull từ đây)
```powershell
git status                       # KIỂM TRA cosign.key KHÔNG có trong danh sách
git add -A
git commit -m "[W10] Lab1 RBAC+Gatekeeper, Lab2 ESO+supply-chain, payments tenant"
git push origin main
```
**Checkpoint:** mở github.com/taitri2004/W10-exercise thấy các thư mục `rbac/ gatekeeper/ eso/ tenants/ apps/`. **Tuyệt đối không thấy `cosign.key`.**

---

## Phần 4 — Tạo cluster (Calico) + ArgoCD + sync
```powershell
minikube start -p w10 --cpus=2 --memory=3000 --driver=docker --cni=calico
kubectl config use-context w10
kubectl create ns argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f argocd/root.yaml
# Giảm tải Gatekeeper (RAM thấp) sau khi nó cài xong vài phút:
kubectl -n gatekeeper-system scale deploy/gatekeeper-controller-manager --replicas=1
```
Xem tiến độ:
```powershell
kubectl -n argocd get applications
```
**Checkpoint:** các Application dần `Synced/Healthy`. (Máy RAM thấp có thể chậm / vài app nặng chưa xanh — không sao, 4 chứng minh dưới đây không cần Prometheus.)

> **Nếu nghẽn RAM:** xoá tạm app nặng không cần cho bài lớn rồi giữ api 1 replica:
> ```powershell
> kubectl -n argocd delete app kube-prometheus-stack argo-rollouts alert analysis eso eso-config policy-controller policies
> kubectl -n demo scale rollout api --replicas=1
> ```
> (Bài lớn chỉ cần: gatekeeper, rbac, api ở demo, payments, payments-app.)

---

## Phần 5 — LẤY 4 CHỨNG MINH BÀI LỚN (phần chấm chính) → chụp vào `evidence/`

### 5.1 RBAC least-privilege (`01-rbac-can-i.png`)
```powershell
kubectl auth can-i create deploy   -n payments --as payments-dev    # yes
kubectl auth can-i create deploy   -n demo     --as payments-dev    # no
kubectl auth can-i update rolebindings -n payments --as payments-dev # no
kubectl auth can-i get secrets     -n payments --as payments-dev    # no
```
**Đạt:** yes / no / no / no.

### 5.2 Quota + LimitRange (`02-quota-limitrange.png`)
```powershell
# Pod thiếu limits VẪN chạy (LimitRange cấp default):
kubectl -n payments run nolimits --image=ghcr.io/taitri2004/w10-api:0.0.1 --labels=owner=team-payments
kubectl -n payments get pod nolimits -o jsonpath='{.spec.containers[0].resources}'   # thấy default

# Pod xin RAM vượt quota -> bị từ chối:
kubectl -n payments run hog --image=ghcr.io/taitri2004/w10-api:0.0.1 --labels=owner=team-payments `
  --overrides='{"spec":{"containers":[{"name":"hog","image":"ghcr.io/taitri2004/w10-api:0.0.1","resources":{"requests":{"memory":"2Gi"},"limits":{"memory":"2Gi"}}}]}}'
# -> Error: exceeded quota: payments-quota
```
**Đạt:** nolimits có default; hog bị `exceeded quota`.

### 5.3 NetworkPolicy cô lập (`03-netpol-block.png`)
```powershell
kubectl apply -f tenants/payments/test/netpol-test-pod.yaml
Start-Sleep 8
kubectl -n payments logs netpol-test
```
**Đạt:** dòng `[1] ... BLOCKED (dung)` (gọi demo/api bị chặn) và `[2] DNS OK`.
> Cần Calico (Phần 4) + app `api` ở demo đang chạy. Dọn: `kubectl -n payments delete pod netpol-test`.

### 5.4 Constraint cũ chặn vi phạm trong tenant (`04-constraint-blocks-tenant.png`) — QUAN TRỌNG NHẤT
```powershell
# app team B hợp lệ chạy xanh:
kubectl -n payments get deploy payments-api
# manifest vi phạm bị CONSTRAINT CŨ chặn (không viết luật mới cho payments):
kubectl apply -f tenants/payments/test/violation-pod.yaml
# -> Error from server: admission webhook "validation.gatekeeper.sh" denied ...
#    [block-latest-tag] :latest ... [require-limits] thiếu cpu/memory ... [require-owner-label] thiếu owner
```
**Đạt:** payments-api Running; bad-pod bị Gatekeeper từ chối.

---

## Phần 6 — Evidence Lab 1 & Lab 2 (làm thêm nếu còn thời gian)

### Lab 1 — RBAC 3 role (`10-rbac-3roles.png`)
```powershell
kubectl auth can-i create deploy -n demo        --as alice   # yes
kubectl auth can-i create deploy -n kube-system --as alice   # no
kubectl auth can-i get pods -A                  --as bob     # yes
kubectl auth can-i delete nodes                 --as carol   # no
```

### Lab 1 — Gatekeeper reject (`11-gatekeeper-reject.png`)
Đã có ở 5.4 (bad-pod). Thêm: deploy pod hợp lệ -> pass.

### Lab 2.1 — ESO rotation (`20-eso-rotation.png`) — cần app `eso`+`eso-config` Synced
```powershell
kubectl -n demo get externalsecret db-creds            # SecretSynced
kubectl -n demo logs -f deploy/db-consumer             # đang in password=v1-rotate-me
# Rotate: sửa eso/secret-store.yaml value -> v2-rotated, version -> v2; commit+push
# Trong < 60s log đổi sang v2-rotated, pod AGE KHÔNG đổi:
kubectl -n demo get pod -l app=db-consumer             # RESTARTS=0, AGE giữ nguyên
```
Repo sạch: `git log -p | Select-String -Pattern "password" ` → không có secret thật.

### Lab 2.2 — Supply chain (`21-supply-chain.png`) — cần Phần 2 Cách B + policy-controller
```powershell
# Sau khi đã ký image (Phần 2B) + bật enforce (Phần 7):
kubectl -n demo run unsigned --image=ghcr.io/taitri2004/w10-api:unsigned --labels=owner=x   # reject (no signature)
# image đã ký (api / payments-api) -> chạy xanh
```

---

## Phần 7 — (Tuỳ chọn) Bật enforce chữ ký — LÀM SAU KHI ĐÃ KÝ
```powershell
# Gắn label SAU khi image đã ký (gắn TRƯỚC sẽ chặn luôn app api):
kubectl label ns demo     policy.sigstore.dev/include=true
kubectl label ns payments policy.sigstore.dev/include=true
```

---

## Phần 8 — Nộp
- Đảm bảo `evidence/` có đủ ảnh 4 chứng minh (Phần 5).
- Commit + push lần cuối (cả ảnh evidence).
- (Nếu mentor chấm qua GitHub) gửi link repo + chỉ rõ `evidence/` và `README.md` (2 câu vì sao).

---

## Ghi chú RAM (máy ~8GB / Docker ~3.7GB)
- `--memory=4000` lỗi vì Docker chỉ ~3.8GB → dùng `--memory=3000`. Nếu được, tăng
  Docker Desktop RAM (Settings → Resources) rồi mới đủ chạy cả stack.
- Không cần chạy tất cả cùng lúc: 4 chứng minh bài lớn chỉ cần
  **Calico + ArgoCD + Gatekeeper(1 replica) + rbac + api(demo) + payments**.
- Gatekeeper mặc định 3 replica → scale 1. ESO/policy-controller chỉ bật khi làm
  evidence Lab 2.
