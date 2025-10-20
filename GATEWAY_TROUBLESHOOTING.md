# Gateway Image Tag 문제 트러블슈팅

## 📌 문제 상황

ArgoCD를 통해 Gateway 서비스를 배포할 때, **의도한 이미지 태그(`fba6ef9`)가 아닌 `latest` 태그로 계속 배포**되는 문제가 발생했습니다.

### 증상
- ArgoCD Parameters에서 `gateway.deployment.image.tag`를 `fba6ef9`로 설정
- 실제 배포된 Pod는 `gateway:latest` 이미지를 사용
- Helm Chart 업데이트 후에도 변경사항이 반영되지 않음

---

## 🔍 원인 분석

### 1. Helm Template 구조 차이

Gateway 서비스의 `deployment.yaml` 템플릿이 다른 서비스들과 **다른 경로의 값을 참조**하고 있었습니다.

#### Gateway Deployment Template
```yaml:charts/loventure/charts/gateway/templates/deployment.yaml
# 라인 34
image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
➡️ `.Values.image.tag` 경로 참조

#### 다른 서비스들 (Auth/Content/Course/Territory/AI)
```yaml
# 예: auth-service/templates/deployment.yaml
image: "{{ .Values.global.imageRegistry }}/{{ .Values.deployment.image.repository }}:{{ .Values.deployment.image.tag }}"
```
➡️ `.Values.deployment.image.tag` 경로 참조

### 2. values.yaml 구조 문제

Gateway 설정이 **잘못된 위치**에 작성되어 있어, Helm 템플릿이 참조하는 경로와 실제 값이 저장된 경로가 달랐습니다.

---

## ❌ 변경 전 Manifest 파일

### 문제가 있던 values.yaml 구조

```yaml:charts/loventure/values.yaml
# Gateway Service Configuration
gateway:
  replicaCount: 1
  image:
    repository: gateway
    tag: latest              # ❌ 문제 1: deployment.yaml이 실제로 참조하는 경로
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 8080
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  # ... 중간 설정 생략 ...
  
  # JWT Secret for Gateway Service
  jwtSecret: "" # Set via ArgoCD UI: gateway.jwtSecret
  
  # Gateway deployment configuration with latest image tag from main
  deployment:
    image:
      tag: "fba6ef9"         # ❌ 문제 2: 사용되지 않는 값 (deployment.yaml이 이 경로를 참조하지 않음)

# ... 하단 생략 ...

# 파일 맨 아래에 추가 중복 설정
# gateway-service:           # ❌ 문제 3: 또 다른 중복 설정
#   deployment:
#     image:
#       tag: cb446c3

territory-service:           # ❌ 문제 4: 중복된 territory-service 설정
  deployment:
    image:
      tag: 4d1e54f
```

### 문제점 요약

| 문제 | 설명 | 영향 |
|------|------|------|
| **문제 1** | `gateway.image.tag: latest`가 실제로 사용됨 | Gateway가 항상 `latest` 태그로 배포 |
| **문제 2** | `gateway.deployment.image.tag: "fba6ef9"`가 무시됨 | ArgoCD Parameters 변경이 적용되지 않음 |
| **문제 3** | 주석 처리된 `gateway-service` 설정 | 혼란 초래 |
| **문제 4** | 중복된 `territory-service` 설정 | 유지보수 어려움 |

---

## ✅ 변경 후 Manifest 파일

### 수정된 values.yaml 구조

```yaml:charts/loventure/values.yaml
# Gateway Service Configuration
gateway:
  replicaCount: 1
  image:
    repository: gateway
    tag: "fba6ef9"           # ✅ 수정: 올바른 태그로 변경
    pullPolicy: Always
  service:
    type: ClusterIP
    port: 8080
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  # ... 중간 설정 생략 ...
  
  # JWT Secret for Gateway Service
  jwtSecret: "" # Set via ArgoCD UI: gateway.jwtSecret
  # ✅ 수정: 사용되지 않는 deployment.image.tag 제거

# Auth Service Configuration
auth-service:
  # ... 생략 ...

# ... 중간 생략 ...

# Territory Service Configuration
territory-service:
  service:
    name: territory-service
    type: ClusterIP
    port: 8084
    targetPort: 8084
  deployment:
    replicaCount: 1
    image:
      repository: territory-service
      tag: "4d1e54f"         # ✅ 수정: latest에서 특정 태그로 변경
      pullPolicy: IfNotPresent
    # ... 생략 ...
  postgres:
    enabled: true
    image:
      registry: docker.io
      repository: postgis/postgis
      tag: "14-3.4"
    # ... 생략 ...

# ✅ 수정: 맨 아래 중복 설정 모두 제거
```

### 수정 사항 요약

| 수정 항목 | 변경 전 | 변경 후 | 이유 |
|----------|---------|---------|------|
| `gateway.image.tag` | `latest` | `"fba6ef9"` | deployment.yaml이 참조하는 올바른 경로에 정확한 태그 설정 |
| `gateway.deployment.image.tag` | `"fba6ef9"` (존재) | 삭제됨 | Gateway 템플릿이 이 경로를 참조하지 않아 불필요 |
| `territory-service.deployment.image.tag` (상단) | `"latest"` | `"4d1e54f"` | 특정 버전으로 고정 |
| 하단 중복 설정 | 존재 | 삭제됨 | 중복 제거로 명확성 향상 |

---

## 🎯 해결 방법

### Step 1: Gateway 이미지 태그 수정

```bash
# charts/loventure/values.yaml 수정
# 라인 11 변경
- tag: latest
+ tag: "fba6ef9"
```

### Step 2: 사용되지 않는 설정 제거

```bash
# 라인 90-93 제거
- # Gateway deployment configuration with latest image tag from main
- deployment:
-   image:
-     tag: "fba6ef9"
```

### Step 3: Territory Service 태그 수정

```bash
# 라인 280 변경
- tag: "latest"
+ tag: "4d1e54f"
```

### Step 4: 하단 중복 설정 제거

```bash
# 라인 309-316 제거
- # gateway-service:
- #   deployment:
- #     image:
- #       tag: cb446c3
- territory-service:
-   deployment:
-     image:
-       tag: 4d1e54f
```

### Step 5: Git Commit & Push

```bash
cd /path/to/PitterPatter-Manifest
git add charts/loventure/values.yaml
git commit -m "fix: Gateway 이미지 태그 경로 수정 및 중복 설정 제거"
git push origin main
```

### Step 6: ArgoCD Sync

```bash
# ArgoCD UI에서 또는 CLI로
argocd app sync loventure-prod

# 또는 kubectl로 확인
kubectl get pods -n loventure-app -l app.kubernetes.io/name=gateway
kubectl describe pod <gateway-pod-name> -n loventure-app | grep Image:
```

---

## 🚨 ArgoCD에서 반영되지 않았던 이유

### 1. Helm Template 경로 불일치

ArgoCD는 Git 저장소의 Helm Chart를 정확히 렌더링합니다. 문제는:

```
ArgoCD가 본 것:
  gateway.image.tag = "latest"  ← deployment.yaml이 이 값을 사용
  gateway.deployment.image.tag = "fba6ef9"  ← 아무도 참조하지 않음
```

**결과**: ArgoCD Parameters에서 `gateway.deployment.image.tag`를 아무리 변경해도, 실제 deployment는 `gateway.image.tag (latest)`를 계속 사용했습니다.

### 2. Values Override 우선순위 문제

Helm의 값 우선순위:
```
1. ArgoCD Parameters (최우선)
2. values.yaml (기본값)
3. Chart.yaml의 appVersion (fallback)
```

하지만 **잘못된 경로**에 Parameters를 설정하면:
- ArgoCD Parameters: `gateway.deployment.image.tag = "fba6ef9"` ← 참조 안됨
- values.yaml: `gateway.image.tag = "latest"` ← 실제로 사용됨
- **결과**: `latest` 태그로 배포

### 3. Helm Diff가 보여주지 않은 이유

```bash
# ArgoCD에서 Helm Diff를 실행하면
helm template . -f values.yaml

# deployment.yaml이 렌더링 되는 방식:
image: "docker.io/pitterpetter/gateway:{{ .Values.image.tag }}"
# .Values.deployment.image.tag는 템플릿에서 참조되지 않아 Diff에 나타나지 않음
```

### 4. 서비스별 Template 구조 차이

| 서비스 | Template 경로 | 올바른 values.yaml 경로 |
|--------|--------------|------------------------|
| Gateway | `.Values.image.tag` | `gateway.image.tag` |
| Auth | `.Values.deployment.image.tag` | `auth-service.deployment.image.tag` |
| Content | `.Values.deployment.image.tag` | `content-service.deployment.image.tag` |
| Course | `.Values.deployment.image.tag` | `course-service.deployment.image.tag` |
| Territory | `.Values.deployment.image.tag` | `territory-service.deployment.image.tag` |
| AI | `.Values.deployment.image.tag` | `ai-service.deployment.image.tag` |

➡️ **Gateway만 구조가 달라서 헷갈림 발생**

---

## 🔧 검증 방법

### 1. Helm Template 렌더링 확인

```bash
cd charts/loventure
helm template loventure-prod . \
  --set gateway.image.tag=fba6ef9 \
  | grep -A 5 "kind: Deployment" \
  | grep -A 20 "name: loventure-prod-gateway"
```

**예상 결과**:
```yaml
containers:
- name: gateway
  image: "docker.io/pitterpetter/gateway:fba6ef9"  # ✅ 올바른 태그
```

### 2. ArgoCD에서 실제 적용된 Manifest 확인

```bash
# ArgoCD CLI
argocd app manifests loventure-prod | grep -A 10 "kind: Deployment" | grep gateway -A 10

# kubectl로 실행 중인 Pod 확인
kubectl get pod -n loventure-app -l app.kubernetes.io/name=gateway -o yaml | grep image:
```

### 3. Git Commit History 확인

```bash
git log --oneline --all --graph -- charts/loventure/values.yaml
git diff HEAD~1 HEAD -- charts/loventure/values.yaml
```

---

## 📚 교훈 및 Best Practices

### 1. Helm Template과 Values 경로 일치 확인
- **항상** `templates/*.yaml`에서 참조하는 경로를 먼저 확인
- `values.yaml`에 값을 작성할 때 정확한 경로에 작성

### 2. 서비스별 Template 구조 통일 고려
현재 Gateway만 다른 구조를 사용하고 있어 혼란 발생:
```yaml
# 옵션 1: Gateway를 다른 서비스와 동일하게 수정 (권장)
gateway/templates/deployment.yaml:
  image: "{{ .Values.deployment.image.repository }}:{{ .Values.deployment.image.tag }}"

values.yaml:
  gateway:
    deployment:
      image:
        tag: "fba6ef9"

# 옵션 2: 현재 구조 유지하되 문서화 강화
```

### 3. ArgoCD Parameters 검증
- Parameters 설정 후 **반드시 App Details에서 렌더링된 Manifest 확인**
- `helm template` 로컬 테스트로 사전 검증

### 4. 중복 설정 제거
- 동일한 값을 여러 곳에 정의하지 않기
- 한 곳에서만 정의하고 참조하도록 구조화

### 5. Git Branch 전략
- `main` 브랜치: 프로덕션 배포용 (검증된 설정만)
- `feature/*` 브랜치: 개발/테스트용
- ArgoCD `targetRevision`과 현재 작업 브랜치 일치 확인

---

## 🔄 재발 방지 체크리스트

배포 전 확인사항:

- [ ] `templates/*.yaml`에서 참조하는 `.Values` 경로 확인
- [ ] `values.yaml`에 해당 경로에 값이 올바르게 정의되어 있는지 확인
- [ ] `helm template` 로컬 렌더링 테스트
- [ ] 중복 설정이 없는지 확인 (`grep -n "service-name:" values.yaml`)
- [ ] Git에 변경사항 커밋 후 Push
- [ ] ArgoCD에서 Sync 전 Diff 확인
- [ ] 배포 후 Pod Image 태그 검증 (`kubectl describe pod`)

---

## 📞 관련 문서

- [Helm Values Files](https://helm.sh/docs/chart_template_guide/values_files/)
- [ArgoCD Parameters Override](https://argo-cd.readthedocs.io/en/stable/user-guide/parameters/)
- [Kubernetes Deployment Best Practices](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**작성일**: 2025-10-18  
**작성자**: PitterPatter DevOps Team  
**관련 이슈**: Gateway Image Tag Mismatch (main 브랜치)

