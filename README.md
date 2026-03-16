# plask-helm — Kubernetes Helm Charts

Plask 플랫폼의 Helm 차트 및 ArgoCD 설정 모음입니다.

---

## 레포지토리 구조

```
plask-helm/
├── apps/                        # ArgoCD Application 매니페스트
│   ├── plask-project.yaml       # ArgoCD AppProject
│   ├── app-infra.yaml           # 외부 서비스 (DB, Redis) + 네임스페이스
│   ├── app-backend.yaml         # Backend API
│   ├── app-worker.yaml          # Order Queue Worker
│   └── app-frontend.yaml        # Frontend (React + Nginx)
│
├── charts/
│   ├── infra/                   # 외부 서비스 ExternalName + Secrets
│   ├── backend/                 # Backend Deployment, Service, HPA, ServiceMonitor
│   ├── worker/                  # Worker Deployment
│   └── frontend/                # Frontend Deployment, Ingress, HPA, nginx ConfigMap
│
├── monitoring/
│   └── grafana-dashboard.yaml   # Plask 전용 Grafana 대시보드
│
└── docs/
    └── redis-vm-setup.md        # Redis VM(10.84.101.160) 듀얼 인스턴스 설정
```

---

## 인프라 구성

```
노트북 1 — Kubernetes 클러스터
  CP   10.84.101.100    (control-plane)
  W1   10.84.101.110  ┐
  W2   10.84.101.120  ├── Worker Nodes (app 배포)
  W3   10.84.101.130  ┘

노트북 2 — 외부 서비스
  DB    10.84.101.150  PostgreSQL 5432
  Redis 10.84.101.160  Redis-Session :6379 / Redis-Queue :6380
```

---

## 빠른 시작

### 1. GitHub 설정

각 레포지토리 생성 후 **`YOUR_GITHUB_ORG`** 를 실제 org/user명으로 교체:

```bash
# apps/*.yaml 및 charts/infra/values.yaml 에서 일괄 교체
grep -rl "YOUR_GITHUB_ORG" . | xargs sed -i 's/YOUR_GITHUB_ORG/실제org명/g'
```

### 2. Redis VM 설정

`docs/redis-vm-setup.md` 참고 — 10.84.101.160에 Redis 2개 인스턴스 설치

### 3. ArgoCD에 레포 등록

```bash
# ArgoCD CLI로 레포 추가
argocd repo add https://github.com/YOUR_GITHUB_ORG/plask-helm \
  --username YOUR_GITHUB_USER \
  --password YOUR_PAT_TOKEN
```

### 4. Helm values 설정 (infra)

`charts/infra/values.yaml` 수정:

```yaml
postgres:
  password: "실제_DB_패스워드"   # ← 변경 필수

secrets:
  jwtSecret: "랜덤_시크릿_키"    # ← 변경 필수
  adminPassword: "관리자_패스워드" # ← 변경 필수
  smtpUser: "이메일@gmail.com"   # ← SMTP 설정 시
  smtpPass: "앱_비밀번호"
```

### 5. ArgoCD AppProject + Applications 등록

```bash
kubectl apply -f apps/plask-project.yaml

# infra를 가장 먼저 배포 (namespace + secret 생성)
kubectl apply -f apps/app-infra.yaml

# 나머지
kubectl apply -f apps/app-backend.yaml
kubectl apply -f apps/app-worker.yaml
kubectl apply -f apps/app-frontend.yaml
```

### 6. ghcr.io 이미지 Pull Secret (private 레포 시)

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USER \
  --docker-password=YOUR_PAT_TOKEN \
  -n plask
```

그 후 각 chart의 `values.yaml`에 추가:

```yaml
imagePullSecrets:
  - name: ghcr-secret
```

### 7. Grafana 대시보드 등록

```bash
kubectl apply -f monitoring/grafana-dashboard.yaml
```

---

## CI/CD 흐름

```
plask-frontend repo (push to main)
  └── GitHub Actions frontend-ci.yml
        ├── Docker build & push → ghcr.io/ORG/plask-frontend:sha-XXXXXXX
        └── yq 로 charts/frontend/values.yaml image.tag 업데이트
              └── push to plask-helm repo
                    └── ArgoCD 감지 → plask-frontend Application 자동 sync

plask-backend repo (push to main)
  └── GitHub Actions backend-ci.yml
        ├── backend image push → ghcr.io/ORG/plask-backend:sha-XXXXXXX
        ├── worker image push  → ghcr.io/ORG/plask-worker:sha-XXXXXXX
        └── charts/backend/values.yaml + charts/worker/values.yaml 업데이트
              └── ArgoCD 자동 sync
```

**필요한 GitHub Secrets (frontend/backend 레포에 각각 설정):**

| Secret 이름 | 설명 |
|------------|------|
| `HELM_REPO_TOKEN` | plask-helm 레포에 push 권한이 있는 GitHub PAT |

---

## 배포 순서 (Sync Wave)

ArgoCD가 다음 순서로 배포합니다:

1. **plask-infra** — Namespace, ExternalName Services, Secret 생성
2. **plask-backend** — Backend Deployment (Secret 참조)
3. **plask-worker** — Worker Deployment (Secret 참조)
4. **plask-frontend** — Frontend Deployment + Ingress

---

## Helm Values 핵심 항목

### infra/values.yaml

| 키 | 설명 | 기본값 |
|----|------|--------|
| `postgres.host` | PostgreSQL VM IP | `10.84.101.150` |
| `redisSession.host` | Redis Session VM IP | `10.84.101.160` |
| `redisQueue.host` | Redis Queue VM IP | `10.84.101.160` |
| `redisQueue.port` | Redis Queue 포트 | `6380` |
| `secrets.jwtSecret` | JWT 서명 키 | **변경 필수** |

### frontend/values.yaml

| 키 | 설명 | 기본값 |
|----|------|--------|
| `image.tag` | 이미지 태그 (CI가 자동 업데이트) | `latest` |
| `ingress.host` | 접속 도메인 | `plask.local` |
| `backend.serviceURL` | nginx가 프록시할 백엔드 URL | `http://plask-backend:3002` |

---

## 모니터링

Prometheus + Grafana가 이미 설치된 상태 기준:

- **ServiceMonitor** — `charts/backend/templates/servicemonitor.yaml`
  - `/metrics` 엔드포인트 스크래핑 (prom-client 기반)
  - label: `release: prometheus` (kube-prometheus-stack 기본값)

- **Grafana Dashboard** — `monitoring/grafana-dashboard.yaml`
  - ConfigMap label `grafana_dashboard: "1"` 로 자동 임포트
  - HTTP 요청수, 응답시간 P99, 에러율, Pod CPU/Memory 패널 포함

---

## 접속 확인

```bash
# Ingress를 통한 접속 (hosts 파일에 등록 필요)
# /etc/hosts: 10.84.101.110  plask.local
curl http://plask.local

# 직접 포트포워딩
kubectl port-forward svc/plask-frontend 8080:80 -n plask
# → http://localhost:8080

# 백엔드 헬스체크
kubectl port-forward svc/plask-backend 3002:3002 -n plask
curl http://localhost:3002/health
```
