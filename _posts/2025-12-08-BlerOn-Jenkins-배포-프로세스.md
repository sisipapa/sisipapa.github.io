---
layout: post
title: "[BlerOn] Backend Jenkins 배포 프로세스"
date: 2025-12-08
sitemap: true
hide_last_modified: true
categories:
- dev
- BlerOn
tags:
- jenkins
- cicd
- docker
- blue-green
- spring-boot
- pipeline
---

* toc
{:toc .large-only}

# [BlerOn] Backend Jenkins 배포 프로세스

[BlerOn] Backend Jenkins Pipeline 기반 CI/CD 배포 프로세스를 정리한 문서입니다.  
Development 환경은 Blue-Green 무중단 배포, Production 환경은 다중 WAS 순차 배포 방식을 사용합니다.

---

## 개요

| 항목 | 내용 |
|------|------|
| 빌드 도구 | Gradle 9.1 + OpenJDK-21 |
| 컨테이너 | Docker |
| 이미지 저장소 | GitLab Container Registry |
| 배포 방식 (Dev) | Blue-Green 무중단 배포 |
| 배포 방식 (Prod) | 다중 WAS 순차 배포 |
| 헬스체크 엔드포인트 | `/actuator/health` |

---

## 파이프라인 전체 흐름

```
Jenkins 트리거
    │
    ▼
Stage 0: 환경 변수 설정
    │
    ▼
Stage 1: Checkout (신규 빌드 시)
    │
    ▼
Stage 2: Gradle 빌드 (신규 빌드 시)
    │
    ▼
Stage 3: Docker 이미지 빌드 & Registry Push (신규 빌드 시)
    │
    ▼
Stage 4: Deploy
    ├── Development → Blue-Green 배포
    └── Production  → WAS 서버 순차 배포
    │
    ▼
Post: Docker 이미지 정리
```

---

## Pipeline 옵션

| 옵션 | 설명 |
|------|------|
| `timestamps()` | 모든 로그에 타임스탬프 출력 |
| `buildDiscarder(logRotator(numToKeepStr: '20'))` | 빌드 히스토리 최대 20개 유지 |
| `disableConcurrentBuilds()` | 동시 빌드 방지 |

---

## Jenkins 파라미터

| 파라미터 | 설명 | 기본값 |
|----------|------|--------|
| `ENV` | 배포 환경 (`development` / `production`) | `production` |
| `DEPLOY_IMAGE_TAG` | 롤백 시 기존 이미지 태그 지정 (미입력 시 신규 빌드) | 없음 |

> `DEPLOY_IMAGE_TAG`를 입력하면 빌드 단계를 건너뛰고 해당 이미지로 즉시 배포(롤백)합니다.

---

## Stage별 상세 설명

### Stage 0: 환경 변수 및 인프라 설정

Jenkins 파라미터 `ENV` 값에 따라 환경별 변수를 동적으로 설정합니다.

**공통 설정**

| 변수 | 설명 |
|------|------|
| `GITLAB_REGISTRY_URL` | GitLab Container Registry URL |
| `IMAGE_REPO_NAME` | Docker 이미지 리포지토리명 |
| `HEALTH_PATH` | 헬스체크 경로 (`/actuator/health`) |

**Development 환경 전용 설정**

| 변수 | 설명 |
|------|------|
| `SLOT_A_NAME / SLOT_B_NAME` | Blue-Green 슬롯 컨테이너명 |
| `SLOT_A_PORT / SLOT_B_PORT` | 슬롯별 포트 (`18081`, `18082`) |
| `ACTIVE_SLOT_FILE` | 현재 활성 슬롯 기록 파일 경로 |
| `JAVA_OPTS_ENV` | JVM 옵션 (`-Xmx8g`) |
| `HEALTH_CHECK_TIMEOUT` | 헬스체크 최대 시도 횟수 |
| `NGINX_UPSTREAM_FILE` | Nginx upstream 설정 파일 경로 |

**Production 환경 전용 설정**

| 변수 | 설명 |
|------|------|
| `DEPLOY_SERVERS` | 배포 대상 WAS 서버 목록 (콤마 구분) |
| `CONTAINER_NAME` | 컨테이너명 |
| `CONTAINER_PORT` | 컨테이너 포트 (`8080`) |

---

### Stage 1: Checkout

- `DEPLOY_IMAGE_TAG` 미입력 시에만 실행 (신규 빌드 모드)
- `checkout scm` 으로 현재 브랜치 소스 코드 체크아웃

---

### Stage 2: Build (Gradle)

- `DEPLOY_IMAGE_TAG` 미입력 시에만 실행
- 명령어: `gradle clean bootJar -x test`
  - 테스트 제외 빌드
  - `bootJar` 산출물 생성 (`build/libs/*.jar`)

---

### Stage 3: Docker 이미지 빌드 & Push

- `DEPLOY_IMAGE_TAG` 미입력 시에만 실행
- 이미지 태그: `{BUILD_NUMBER}-{TARGET_ENV}` (환경별 충돌 방지)

**처리 흐름**

```
1. build/libs/ 에서 bootJar 산출물 탐색 (-plain 제외)
2. app.jar 로 복사
3. docker build -t {이미지태그} --build-arg JAR_FILE=app.jar .
4. GitLab Registry 로그인
5. docker push {이미지태그}
6. docker logout
```

---

### Stage 4-A: Development 배포 (Blue-Green)

무중단 배포를 위해 Blue(A슬롯) / Green(B슬롯) 두 컨테이너를 교대로 사용합니다.

**배포 흐름**

```
1. GitLab Registry 로그인 → Docker 이미지 Pull (로컬 없을 때만)
       │
       ▼
2. active.slot 파일 읽기 → 현재 슬롯 확인 (A/B)
       │
       ▼
3. 비활성 슬롯(New Slot)에 새 컨테이너 실행
   - 포트: 18081(A슬롯) 또는 18082(B슬롯)
   - Spring Profile: dev
   - 로그 볼륨 마운트
       │
       ▼
4. 헬스체크 (최대 30회 × 2초 간격)
   - 성공 → 다음 단계
   - 실패 → 새 컨테이너 제거, 이전 컨테이너 유지, 배포 실패
       │
       ▼
5. Nginx upstream 파일 업데이트 → nginx -t 검증 → nginx reload
       │
       ▼
6. active.slot 파일 갱신 (새 슬롯으로 업데이트)
       │
       ▼
7. 구 컨테이너(이전 슬롯) 정지 & 제거
       │
       ▼
8. 불필요 이미지 정리 (docker image prune)
```

**슬롯 전환 예시**

| 현재 활성 슬롯 | 신규 배포 슬롯 | Nginx 전환 후 |
|--------------|-------------|-------------|
| A (18081) | B (18082) | B (18082)로 트래픽 전환 |
| B (18082) | A (18081) | A (18081)로 트래픽 전환 |

---

### Stage 4-B: Production 배포 (WAS 순차 배포)

여러 WAS 서버에 순서대로 배포합니다. 한 서버가 실패하면 중단됩니다.

**배포 흐름**

```
1. Jenkins에서 환경변수 파일(.env) 임시 생성
   - DB, Redis, JWT, AWS, 결제, SNS, CDN 등 민감정보 포함
   - mktemp로 고유 파일명 생성
       │
       ▼
2. SCP로 배포 대상 서버에 .env 파일 전송
       │
       ▼
3. Jenkins 로컬의 .env 파일 즉시 삭제
       │
       ▼
4. SSH로 대상 서버 접속 → 배포 실행
   a. GitLab Registry 로그인 → 이미지 Pull (로컬 없을 때만)
   b. 기존 컨테이너 정지 & 제거
   c. 새 컨테이너 실행 (--env-file 사용)
   d. 헬스체크 (최대 20회 × 5초 간격)
   e. .env 파일 삭제 (trap EXIT)
   f. 불필요 이미지 정리
       │
       ▼
5. 다음 WAS 서버 반복 (WAS1 → WAS2 → ...)
```

**Production 환경변수 파일에 포함되는 시크릿 목록**

| 분류 | 항목 |
|------|------|
| Spring | SPRING_PROFILES_ACTIVE, TZ |
| DB | DB_MASTER (username/password), DB_SLAVE (username/password) |
| Redis | REDIS (username/password) |
| 이메일 | EMAIL_AUTH (id/password) |
| 보안 | JWT_SECRET, ENCRYPTION_KEY, HMAC_KEY |
| AWS | AWS_ACCESS_KEY, AWS_SECRET_KEY |
| SMS/알림 | SMS_API_TOKEN, ALIM_API_TOKEN |
| SNS 로그인 | NAVER / KAKAO / GOOGLE (client_id/secret) |
| 결제 | NICEPAY (merchant_id/secret_key) |
| CDN | CDN / VOD / DOWNLOAD (user/password, secret_key) |
| 기타 | EXTERNAL_API_TOKEN, NOTIFICATION_TOKEN, CHANNEL_TALK_KEY |

> 모든 시크릿은 Jenkins Credentials에 등록된 값을 사용하며, 소스코드에 직접 포함되지 않습니다.

---

### Post: 파이프라인 종료 처리

| 상태 | 처리 내용 |
|------|----------|
| 항상 | GitLab Registry 로그아웃, `docker image prune -f` |
| 성공 | 빌드에 사용된 Docker 이미지 삭제 (`docker rmi`) |
| 실패 | 실패 로그 출력 |

---

## 롤백 방법

이미지 태그를 지정해 이전 버전으로 즉시 배포합니다.

1. Jenkins 파이프라인 실행
2. `DEPLOY_IMAGE_TAG` 파라미터에 롤백할 이미지 태그 입력
   - 예: `42-production`
3. 빌드/이미지 생성 단계를 건너뛰고 해당 이미지로 바로 배포

---

## 환경별 비교

| 항목 | Development | Production |
|------|------------|-----------|
| 배포 방식 | Blue-Green (무중단) | 순차 배포 |
| 슬롯 수 | 2개 (A/B) | 서버 대수만큼 |
| Nginx 전환 | 있음 (upstream 파일 갱신) | 없음 (로드밸런서 외부 관리) |
| 헬스체크 간격 | 2초 | 5초 |
| 헬스체크 최대 횟수 | 30회 | 20회 |
| JVM 옵션 | `-Xmx8g` | 없음 (컨테이너 기본값) |
| 시크릿 주입 방식 | 환경변수 직접 (`-e`) | env 파일 (`--env-file`) |
| Spring Profile | `dev` | `production` |

---

## 보안 고려사항

- **Jenkins Credentials**: 모든 민감정보(DB, JWT, API 키 등)는 Jenkins Credentials에 저장하여 파이프라인에서 참조
- **env 파일 즉시 삭제**: SCP 전송 후 Jenkins 로컬 .env 파일 즉시 삭제, 원격 서버에서도 `trap EXIT`으로 자동 삭제
- **이미지 태그 구분**: `{BUILD_NUMBER}-{ENV}` 형식으로 환경별 이미지 충돌 방지
- **StrictHostKeyChecking=no**: SSH 자동화를 위해 비활성화 (내부망 전용)
- **Registry 로그아웃**: 배포 완료 후 즉시 로그아웃 처리
