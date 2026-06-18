# Runbook: Secret rotation lỗi / app dùng giá trị cũ

- **Mức độ:** SEV-2

## Triệu chứng
- Đã đổi value ở store (AWS / fake) nhưng app vẫn dùng giá trị cũ, hoặc
  `ExternalSecret` status `SecretSyncedError`.

## Chẩn đoán
```bash
kubectl -n demo get externalsecret db-creds
kubectl -n demo describe externalsecret db-creds      # reason/lỗi auth
kubectl -n demo get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

## Xử lý
1. **App dùng env thay vì volume**: env chốt lúc start → rotate không ăn tới khi
   restart. Sửa: mount Secret dạng **volume** (như `eso/consumer.yaml`). Mitigate:
   `kubectl -n demo rollout restart deploy/db-consumer`.
2. **SecretSyncedError**: kiểm auth store (AWS creds/region/key đúng không).
3. **Ép sync ngay** (không chờ refreshInterval):
   `kubectl -n demo annotate externalsecret db-creds force-sync=$(date +%s) --overwrite`
4. Xác nhận: file mount đổi sang giá trị mới, pod `AGE` không đổi.

## Leo thang
- Secret prod sai > 15 phút → báo owner dịch vụ.
