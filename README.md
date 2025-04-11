# GCP 기반 CI/CD 파이프라인 및 서버 운영 구조

## 개요
본 저장소는 Google Cloud Platform(GCP) 환경에서 운영되는 서비스의 **CI/CD 파이프라인 자동화**, **서버 무중단 배포**, **부하 분산**, **WAS/Web 계층 구조 관리**를 목적으로 구성되었습니다. 백엔드 서비스 운영자가 GCP 인프라를 기반으로 한 CI/CD 관리와 서비스 배포를 효과적으로 수행할 수 있도록 작성되었습니다.

---

## 시스템 아키텍처

![GCP Infrastructure Diagram](docs/서버구성도.png)

- **Cloud Armor**: Web Browser 및 Mobile App으로부터의 트래픽 필터링 및 보안 처리
- **HTTP Load Balancer**: 외부 요청에 대한 트래픽 분산 처리
- **Web Server (WEB1, WEB2)**: 외부 요청 처리, 내부 Load Balancer로 포워딩
- **Internal HTTP Load Balancer**: WAS 간 트래픽 분산
- **WAS (WAS1, WAS2)**: 주요 애플리케이션 로직 처리
- **Search Engine**: 내부 검색 엔진 시스템
- **Cloud SQL**: 주요 DB 데이터 저장소
- **배치 서버**: 스케줄 기반 데이터 처리 및 백엔드 작업

---

## Google Cloud 환경 서버 구성

- **WEB 서버 (WEB1, WEB2)**: HTTP 요청을 처리하는 프론트엔드 서버로, Cloud Load Balancer에 의해 트래픽이 분산됩니다.
- **WAS 서버 (WAS1, WAS2)**: 내부 HTTP Load Balancer를 통해 분산된 트래픽을 처리하며, 실질적인 백엔드 로직과 데이터 연동을 수행합니다.
- **Search Engine 서버**: 데이터 검색 및 인덱싱 로직을 수행
- **Cloud SQL**: GCP에서 제공하는 RDBMS 서비스로 주요 비즈니스 데이터를 저장

---

## CI/CD 파이프라인 구성 (Google Cloud Build 기반)

### 1. Cloud Source Repositories 연동
- GitHub 저장소 또는 Cloud Source Repositories에 `master` 브랜치 푸시 이벤트를 기준으로 CI/CD 트리거 발생

### 2. Cloud Build Trigger 구조
- 변경사항이 `master` 브랜치에 push 되면, **Google Cloud Build Trigger**가 감지
- 승인 대기 후 수동 승인 시 `.yaml` 빌드 파일을 기반으로 자동 배포 수행

### 3. cloudbuild.yaml 배포 파이프라인 예시
```yaml
steps:
  - name: 'gcr.io/cloud-builders/gcloud'
    id: WAS gce 1
    entrypoint: /bin/sh
    args:
      - '-c'
      - |
        set -x && \
        gcloud compute ssh gw-was-prod-1 --zone=asia-northeast3-a --tunnel-through-iap \
        --command='sudo /bin/sh /service/tomcatBuild/buildAndDeploy.sh'

  - name: 'gcr.io/cloud-builders/gcloud'
    id: WAS gce 2
    waitFor:
      - WAS gce 1
    entrypoint: /bin/sh
    args:
      - '-c'
      - |
        set -x && \
        gcloud compute ssh gw-was-prod-2 --zone=asia-northeast3-a --tunnel-through-iap \
        --command='sudo /bin/sh /service/tomcatBuild/buildAndDeploy.sh'
```

### 4. 무중단 배포 시나리오
- `WAS1` 서버에 먼저 배포를 수행
- `WAS1` 배포가 완료되면 `WAS2`로 배포가 이어짐
- 이중 서버 구조를 통해 트래픽을 유지하며 무중단 배포 가능

---

## 주요 특성

- ✅ **Cloud Load Balancing을 이용한 트래픽 분산**
- ✅ **Cloud Build + Compute Engine SSH를 통한 서버 배포 자동화**
- ✅ **이중 WAS 서버 구조를 통한 무중단 배포 보장**

