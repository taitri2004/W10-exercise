# Evidence — ảnh/log chứng minh (chụp khi chạy live)

Đặt ảnh/log vào đây theo tên gợi ý. 4 chứng minh của bài lớn là phần chấm chính.

## Bài lớn (bắt buộc — 4 chứng minh)
- `01-rbac-can-i.png` — `payments-dev`: tạo deploy trong `payments`=yes · trong
  `demo`=no · sửa rolebinding=no · đọc secret=no
  ```bash
  kubectl auth can-i create deploy -n payments --as payments-dev   # yes
  kubectl auth can-i create deploy -n demo     --as payments-dev   # no
  kubectl auth can-i update rolebindings -n payments --as payments-dev   # no
  kubectl auth can-i get secrets -n payments   --as payments-dev   # no
  ```
- `02-quota-limitrange.png` — pod vượt quota bị reject · pod thiếu limits chạy nhờ
  LimitRange (`kubectl -n payments describe quota payments-quota`)
- `03-netpol-block.png` — `kubectl apply -f tenants/payments/test/netpol-test-pod.yaml`
  rồi `kubectl -n payments logs netpol-test` → demo/api BLOCKED, DNS OK
- `04-constraint-blocks-tenant.png` — `kubectl apply -f tenants/payments/test/violation-pod.yaml`
  → bị constraint CŨ chặn · app `payments-api` hợp lệ chạy xanh

## Lab 1
- `10-rbac-3roles.png` — 4 lệnh `auth can-i` (alice/bob/carol) đúng kỳ vọng
- `11-gatekeeper-reject.png` — 4 pod vi phạm reject + 1 hợp lệ pass

## Lab 2
- `20-eso-rotation.png` — đổi value → Secret đổi < 60s · pod `AGE` không đổi
- `21-supply-chain.png` — CI đỏ khi CVE HIGH · image chưa ký reject · đã ký pass

## ArgoCD
- `30-argocd-synced.png` — tất cả Application `Synced/Healthy`
