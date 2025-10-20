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

## 핵심 특징

### Development 환경
- **개별 데이터베이스**: 각 서비스마다 독립적인 PostgreSQL 인스턴스
- **격리된 환경**: 개발 및 테스트를 위한 완전한 서비스 격리
- **카카오 클라우드**: 비용 효율적인 개발 환경 제공

### Production 환경
- **관리형 데이터베이스**: 모든 서비스가 하나의 고가용성 PostgreSQL 공유
- **고가용성 (HA)**: 다중 AZ 배포로 99.95% 가용성 보장
- **재해복구 (DR)**: 자동 백업 및 크로스 리전 복제로 데이터 보호
- **CDC 파이프라인**: Change Data Capture를 통한 실시간 데이터 동기화 및 분석
- **확장성**: GCP의 관리형 서비스로 자동 스케일링 및 백업
- **외부 접근**: Ingress를 통한 안전한 외부 API 접근 (api.loventure.us)
- **프로덕션급 안정성**: 모니터링, 로깅, 알림 시스템 통합

### 공통 특징
- **완전한 GitOps**: ArgoCD가 자기 자신을 Git에서 관리
- **Helm 기반**: 모든 애플리케이션을 Helm 차트로 관리
- **자동 배포**: Git push만으로 자동 배포
- **RBAC 보안**: ArgoCD default 프로젝트로 권한 관리
- **일관성**: ArgoCD와 애플리케이션을 동일한 방식으로 관리


