# Loventure GitOps Repository

이 프로젝트는 ArgoCD와 GitHub Actions를 통한 GitOps 기반 CI/CD 파이프라인을 지원합니다.

## 🏗️ 아키텍처

### 서비스 구성
- **auth-service**: 인증 서비스 (Spring Boot + PostgreSQL)
- **content-service**: 콘텐츠 관리 서비스 (Spring Boot + PostgreSQL)  
- **course-service**: 코스 관리 서비스 (Spring Boot + PostgreSQL)
- **ai-service**: AI 서비스 (FastAPI)

### 환경 구성
- **develop 브랜치(Staging)**: 카카오 클라우드 쿠버네티스 서비스
- **main 브랜치(Deployment)**: GCP 쿠버네티스 서비스

## 📁 프로젝트 구조

```
PitterPatter-Manifest/
├── apps/                                 # ArgoCD Application 매니페스트
│   └── applications/
│       ├── argocd.yaml                  # ArgoCD 자기 관리 Application
│       ├── loventure-prod.yaml          # Production 환경 루트 Application
│       └── default-project.yaml         # ArgoCD default 프로젝트 설정
├── charts/                              # Helm 차트
│   ├── argo-cd/                         # ArgoCD 설치용 Helm 차트
│   │   ├── Chart.yaml                   # ArgoCD 공식 차트 의존성
│   │   └── values.yaml                  # ArgoCD 설정값
│   └── loventure/                       # 부모 차트
│       ├── Chart.yaml
│       ├── values.yaml                  # Production 설정
│       ├── values-dev.yaml              # Development 설정
│       ├── charts/                      # 서브차트들
│       │   ├── auth-service/            # 인증 서비스 차트
│       │   ├── content-service/         # 콘텐츠 서비스 차트
│       │   ├── course-service/          # 코스 서비스 차트
│       │   └── ai-service/              # AI 서비스 차트
│       └── templates/
│           └── _helpers.tpl
├── BOOTSTRAP_GUIDE.md                   # ArgoCD 부트스트랩 가이드
└── README.md                            # 이 파일
```

## 🔄 CI/CD 파이프라인

### 브랜치 전략
- **`develop` 브랜치** → **카카오 클라우드** (Development/Staging 환경)
- **`main` 브랜치** → **GCP** (Production 환경)

### 배포 흐름

#### 1. Development 환경 배포
```bash
# develop 브랜치에 코드 푸시
git push origin develop
```
→ ArgoCD가 자동으로 감지하여 **카카오 클라우드**에 배포

#### 2. Production 환경 배포
```bash
# main 브랜치에 머지
git merge develop
git push origin main
```
→ ArgoCD가 자동으로 감지하여 **GCP**에 배포



## 🎯 핵심 특징

- **완전한 GitOps**: ArgoCD가 자기 자신을 Git에서 관리
- **Helm 기반**: 모든 애플리케이션을 Helm 차트로 관리
- **GCP 전용**: main 브랜치에서 GCP 프로덕션 환경으로 배포
- **자동 배포**: Git push만으로 자동 배포
- **RBAC 보안**: ArgoCD default 프로젝트로 권한 관리
- **일관성**: ArgoCD와 애플리케이션을 동일한 방식으로 관리


