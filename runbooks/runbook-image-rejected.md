# Runbook: Pod bị admission từ chối (signature / constraint)

- **Mức độ:** SEV-2 (không deploy được, dịch vụ cũ vẫn chạy)

## Triệu chứng
- `kubectl apply` / ArgoCD báo `admission webhook denied`:
  - Gatekeeper: `... container <api> dùng tag :latest` / `thiếu cpu limit` / ...
  - Sigstore: `... failed policy: require-signed-w10-api: no matching signatures`

## Chẩn đoán
```bash
kubectl get constraints          
kubectl get clusterimagepolicy
kubectl get ns demo --show-labels | grep sigstore
```

## Xử lý
1. **Vi phạm Gatekeeper**: sửa manifest cho hợp lệ (pin tag, thêm limits, non-root,
   label owner). KHÔNG hạ constraint xuống `warn` ở production để "cho qua".
2. **Image chưa ký**: ký image bằng cosign key (xem `signing/README.md`), deploy
   lại đúng tag/digest đã ký. Nếu là CVE chưa vá → exception ADR có thời hạn.
3. **Bẫy thứ tự**: nếu vừa gắn label `policy.sigstore.dev/include=true` mà image
   chưa ký → gỡ label, ký xong gắn lại.

## Leo thang
- Chặn release gấp > 15 phút → báo owner.
