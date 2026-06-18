# W10 - Progressive Delivery with Analysis

GitOps setup for API deployment với Argo Rollouts + AnalysisTemplate.

## Concept

Deploy API với **canary strategy** và **automated analysis**:
- Rollout: 10% → 50% → 100%
- AnalysisTemplate query Prometheus để check success rate ≥ 95%
- Auto rollback nếu analysis fail
- AlertManager gửi email khi có SLO violation

## Requirements

- Docker Desktop
- kubectl
- minikube
- git

## Structure

```
w10/
├── app-api/              # API Rollout manifests
│   ├── rollout.yaml      # Argo Rollout với canary strategy
│   ├── service.yaml      # Service expose API
│   └── servicemonitor.yaml # Prometheus metrics scraper
├── app-analysis/         # Analysis manifests
│   └── analysis-template.yaml # Template phân tích success rate
├── app-alert/            # Alert manifests
│   ├── prometheus-rules.yaml # PrometheusRule cho SLO alerts
│   ├── email-secret.yaml # Gmail password (NOT COMMITTED)
│   └── README.md         # Alert setup guide
├── app-common/           # Common resources
│   └── demo-namespace.yaml # Namespace demo
├── src/                  # Source code
│   └── api/              # Flask API application
├── argocd/
│   ├── apps/             # ArgoCD Application manifests
│   │   ├── app-api.yaml  # Deploy API Rollout
│   │   ├── app-analysis.yaml # Deploy AnalysisTemplate
│   │   ├── app-alert.yaml # Deploy PrometheusRule
│   │   ├── app-common.yaml # Deploy common resources
│   │   ├── k8s-prometheus.yaml # Prometheus + AlertManager
│   │   └── k8s-rollout.yaml # Argo Rollouts controller
│   └── root.yaml         # App of Apps pattern
└── README.md
```

## Quick Start

### 1. Setup Cluster
```bash
minikube start -p w10 --driver=docker
kubectl config use-context w10
```

### 2. Install ArgoCD
```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### 3. Access ArgoCD UI
```bash
# Port forward
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### STEP PHẢI LÀM ĐỂ APP API CHẠY ĐƯỢC
Step 1: Phải build image:
- Dùng Github Action tại `.github/workflows/build-push.yml` để build image.
- Hoặc build local và đẩy lên k8s

Step 2: Phải đổi image name dòng `24` trong file `app-api/rollout.yaml` thành image các bạn đã build

> Note 1: Fork repo thì sẽ không active được Github Action

> Note 2: Nên clone repo template này về sau đó đẩy lên 1 repo của các bạn

> Note 3: Phải đổi đúng image mà các bạn đã build nhé

### 4. Deploy App of Apps
```bash
kubectl apply -f argocd/root.yaml
```

### 5. Setup Email Alert
```bash
# Follow instructions in app-alert/README.md
cp app-alert/email-secret.yaml.example app-alert/email-secret.yaml
kubectl apply -f app-alert/email-secret.yaml
```

## Components

### Core
- **Argo Rollouts**: Progressive delivery controller
- **Prometheus Stack**: Metrics collection + AlertManager
- **API**: Flask application với metrics endpoint

### GitOps Applications
- `app-api`: API Rollout với canary strategy
- `app-analysis`: AnalysisTemplate cho automated validation
- `app-alert`: PrometheusRule cho runtime alerting
- `app-common`: Shared resources (namespace)
- `k8s-prometheus`: Monitoring stack
- `k8s-rollout`: Argo Rollouts controller

## Verify Deployment

### Check Rollout Status
```bash
# Watch rollout progress
kubectl get rollout api -n demo -w

# Check current state
kubectl get rollout api -n demo

# Check pods
kubectl get pods -n demo -l app=api
```

### Check AnalysisRun
```bash
# List analysis runs
kubectl get analysisrun -n demo

# Watch latest analysis
kubectl get analysisrun -n demo --sort-by=.metadata.creationTimestamp | tail -1

# Describe for detailed metrics
kubectl describe analysisrun -n demo <name>
```

### Query Prometheus Metrics
```bash
# Success rate metric
kubectl run test-query --image=curlimages/curl:latest --rm -i --restart=Never -n monitoring -- \
  curl -s 'http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query=api:success_rate:5m'
```

## Test Scenarios (GitOps)

### Test 1: Successful Deployment (Success Rate ≥ 90%)
```bash
# Edit rollout to deploy with no errors
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0"

git add app-api/rollout.yaml
git commit -m "test: deploy with 0% error rate"
git push origin main

# Watch AnalysisRun succeed
kubectl get analysisrun -n demo -w
```

### Test 2: Failed Deployment (Success Rate < 90%)
```bash
# Edit rollout to deploy with 15% error rate
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.15"

git add app-api/rollout.yaml
git commit -m "test: deploy with 15% error rate (should fail)"
git push origin main

# Watch AnalysisRun fail and auto rollback
kubectl get analysisrun -n demo -w
kubectl get rollout api -n demo
```

### Test 3: Trigger SLO Alert Email
```bash
# Edit rollout to set 10% error rate (triggers alert, but passes canary)
nano app-api/rollout.yaml
# Set: ERROR_RATE: "0.10"

git add app-api/rollout.yaml
git commit -m "test: deploy with 10% error rate (90% success)"
git push origin main

# Canary passes (≥90%) but SLO alert fires (below 95%)
# Wait 2-3 minutes, then check email inbox
```


## Configuration Reference

### Sync Waves
ArgoCD applications deploy in order:
- Wave -1: `app-common` (namespace)
- Wave 0: `k8s-prometheus`, `k8s-rollout` (infrastructure)
- Wave 1: `app-analysis`, `app-alert` (configuration)
- Wave 2: `app-api` (application)

## Cleanup

```bash
# Delete ArgoCD applications
kubectl delete -f argocd/root.yaml

# Wait for resources to be cleaned up
kubectl get all -n demo
kubectl get all -n monitoring

# Delete ArgoCD
kubectl delete ns argocd

# Stop minikube
minikube stop -p w10
minikube delete -p w10
```

---

# Bài tập lớn W10 — Onboard team `payments` (multi-tenant an toàn)

Đón team thứ hai `payments` vào cụm đã siết bảo mật (Lab 1–2): cấp "phòng riêng",
guardrail cũ **tự áp** cho team mới, 2 team không gọi qua lại nhau. Tất cả qua GitOps.

## 2 câu "vì sao" (giải thích)
1. **Vì sao guardrail cũ tự áp cho team B mà không viết luật mới?** Gatekeeper
   Constraint và Sigstore ClusterImagePolicy là **cluster-scoped**, áp theo
   *namespace/label/image-glob* chứ không theo team. Constraint match
   `namespaces: [demo, payments]` và webhook soi mọi Pod khớp → chỉ cần thêm ns
   `payments` vào match là kế thừa toàn bộ, không viết constraint mới.
2. **Role/RoleBinding khác ClusterRoleBinding ra sao để giữ cô lập?**
   `Role`+`RoleBinding` **bó trong ns `payments`** nên `payments-dev` chỉ có quyền
   trong `payments`, không với sang `demo`. `ClusterRoleBinding` cấp quyền **toàn
   cụm** (mọi ns) → phá cô lập. Vì vậy tenant dùng RoleBinding namespaced.

## Cấu trúc thêm vào (so với platform W9)
```
rbac/                     # Lab 1.1 — 3 role alice/bob/carol
gatekeeper/templates|constraints/  # Lab 1.2 (4) + 1.3 (custom owner-label)
eso/                      # Lab 2.1 — SecretStore(fake) + ExternalSecret + consumer
signing/ + policies/      # Lab 2.2 — cosign.pub + ClusterImagePolicy
tenants/payments/         # Bài lớn — ns + rbac + quota + netpol (+ test/)
apps/payments/            # Bài lớn — app team B (image đã ký)
argocd/apps/*.yaml        # Application cho tất cả ở trên
runbooks/ · evidence/
```

## Chạy (GitOps)
```bash
# Cluster CẦN Calico để NetworkPolicy enforce
minikube start -p w10 --cpus=2 --memory=4000 --driver=docker --cni=calico
kubectl config use-context w10
# ArgoCD (xem Quick Start phía trên) rồi:
kubectl apply -f argocd/root.yaml
# Ký image + gắn label sigstore SAU khi ký (xem signing/README.md)
```
Thứ tự sync-wave: ns(-1) → controllers gatekeeper/eso/policy-controller(0) →
rbac/alert/analysis(1) → config gatekeeper/eso/policies + api(2) → payments(3) →
payments-app(4).

Bằng chứng + checklist 4 chứng minh: `evidence/README.md`.

> Lưu ý máy RAM thấp: chạy cả stack cùng lúc rất nặng — có thể bật từng nhóm App
> để lấy evidence, không cần đồng thời. Constraint/policy chỉ deny trên ns `demo`
> + `payments` nên không đụng hệ thống.

