# Loventure GitOps Repository

이 프로젝트는 ArgoCD와 GitHub Actions를 통한 GitOps 기반 CI/CD 파이프라인을 지원합니다.

## 아키텍처

### 서비스 구성

#### Development 환경 (develop 브랜치)
- **Gateway Service**: API Gateway (Spring Boot)
- **Auth Service**: 인증 서비스 (Spring Boot + 개별 PostgreSQL)
- **Content Service**: 콘텐츠 관리 서비스 (Spring Boot + 개별 PostgreSQL)
- **Course Service**: 코스 관리 서비스 (Spring Boot + 개별 PostgreSQL)
- **AI Service**: AI 서비스 (FastAPI)
- **Redis**: 캐싱 서비스
- **Territory Service**: 지역 서비스 (Spring Boot + 개별 PostgreSQL)

**특징:**
- 각 서비스마다 독립적인 PostgreSQL 데이터베이스
- 개발 및 테스트 목적의 격리된 환경
- 카카오 클라우드 플랫폼에서 실행

#### Production 환경 (main 브랜치)
- **Gateway Service**: API Gateway (Spring Boot)
- **Auth Service**: 인증 서비스 (Spring Boot + 관리형 PostgreSQL)
- **Content Service**: 콘텐츠 관리 서비스 (Spring Boot + 관리형 PostgreSQL)
- **Course Service**: 코스 관리 서비스 (Spring Boot + 관리형 PostgreSQL)
- **AI Service**: AI 서비스 (FastAPI)
- **Redis**: 캐싱 서비스
- **Territory Service**: 지역 서비스 (Spring Boot + 관리형 PostgreSQL)

**특징:**
- 모든 서비스가 하나의 관리형 PostgreSQL 데이터베이스 공유
- 고가용성 및 확장성을 위한 프로덕션 환경
- Google Cloud Platform에서 실행
- Ingress 설정으로 외부 접근 가능 (api.loventure.us)

## 프로젝트 구조

```
PitterPatter-Manifest/
├── apps/                                 # ArgoCD Application 매니페스트
│   └── applications/
│       ├── argocd.yaml                  # ArgoCD 자기 관리 Application
│       ├── loventure-prod.yaml          # Production 환경 통합 Application (Umbrella Chart)
│       └── default-project.yaml         # ArgoCD default 프로젝트 설정
├── charts/                              # Helm 차트
│   ├── argo-cd/                         # ArgoCD 설치용 Helm 차트
│   │   ├── Chart.yaml                   # ArgoCD 공식 차트 의존성
│   │   └── values.yaml                  # ArgoCD 설정값
│   └── loventure/                       # Umbrella 차트 (모든 서비스 통합)
│       ├── Chart.yaml                   # 모든 서비스 의존성 정의
│       ├── values.yaml                  # Production 환경 통합 설정
│       └── charts/                      # 서브차트들
│           ├── gateway/                 # API Gateway 차트
│           ├── auth-service/            # 인증 서비스 차트
│           ├── content-service/         # 콘텐츠 서비스 차트
│           ├── course-service/          # 코스 서비스 차트
│           ├── ai-service/              # AI 서비스 차트
│           ├── territory-service/       # 지역 서비스 차트
│           └── redis/                   # Redis 캐싱 서비스 차트
├── BOOTSTRAP_GUIDE.md                   # ArgoCD 부트스트랩 가이드
└── README.md                            # 이 파일
```

## CI/CD 파이프라인

### 브랜치 전략
- **`develop` 브랜치** → **카카오 클라우드** (Development 환경 - 개별 DB)
- **`main` 브랜치** → **GCP** (Production 환경 - 관리형 DB)

### 배포 흐름

#### 1. Development 환경 배포
```bash
# develop 브랜치에 코드 푸시
git push origin develop
```
→ ArgoCD가 자동으로 감지하여 **카카오 클라우드**에 배포
→ 각 서비스마다 독립적인 PostgreSQL 데이터베이스 사용

#### 2. Production 환경 배포
```bash
# main 브랜치에 머지
git merge develop
git push origin main
```
→ ArgoCD가 자동으로 감지하여 **GCP**에 배포
→ 모든 서비스가 하나의 관리형 PostgreSQL 데이터베이스 공유

## CI/CD 동작 과정

### Development (develop)
- 트리거: 각 서비스 레포지토리 dev 브랜치 push
- 동작:
  - GitHub Actions가 Docker 이미지 빌드 및 Docker Hub push (태그: {sha}, latest)
  - Manifest 저장소 develop 브랜치의 charts/loventure/values.yaml 내 각 서비스 이미지 태그 업데이트
  - ArgoCD가 develop 브랜치 감시 → 개발 클러스터로 자동 배포
- 환경 변수:
  - 각 서비스별 개별 DB 사용 → values.yaml 또는 Parameters에 URL/USER/PASSWORD 설정

### Production (main)
- 트리거: Manifest main 브랜치 변경 (또는 develop → main 머지)
- 동작:
  - ArgoCD가 main 브랜치 감시 → umbrella 차트(loventure)로 전체 서비스 동기화/배포
  - 관리형 PostgreSQL을 사용하므로 in-cluster DB 설치 없음
- 환경 변수(ArgoCD Parameters로 설정):
  - gateway.jwtSecret
  - auth-service.deployment.env[1..3] (DB URL/USER/PASSWORD)
  - content-service.deployment.env[1..3]
  - course-service.deployment.env[1..3]
  - territory-service.deployment.env[1..3]
  - ai-service.deployment.env[2] (SECRET_KEY)

### 이미지 태그 정책
- Dev: 각 서비스 CI가 {sha}로 태그 빌드/푸시, develop의 values에 반영
- Prod: 필요한 경우 develop의 {sha}를 main에 반영하여 동일 이미지 사용 보장

### 운영 팁
- 변경 후: git push 후 ArgoCD UI에서 loventure-prod Sync 상태 확인
- 강제 동기화: `argocd app sync loventure-prod` 또는 kubectl patch operation 사용
- 롤백: ArgoCD Application의 History에서 원하는 Revision으로 Rollback 가능





