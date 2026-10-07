# 편리햄! – AI 기반 워크스페이스 관리 서비스

## 프로젝트 개요
편리햄은 Mattermost 플랫폼을 확장하여 협업 환경을 더 효율적이고 편리하게 관리할 수 있도록 만든 서비스입니다.
메시지·파일을 실시간으로 수집하고, AI 기반 분류, 고급 검색, 실시간 알림, 추천 기능을 제공합니다.

---

## 주요 기능

### AI 기반 메시지 분석
- 문서 및 메시지 자동 분류 및 라벨링

<img src="https://github.com/user-attachments/assets/849b4c93-fbc0-49d7-922a-e917b8ea1d54" height="400" weight ="600"/>


### 실시간 알림 (SSE & Chatbot)
- 키워드 알림
- 마감 임박 알림
- 선호 포지션 & 기술 스택 구직 정보 알림
- 게시글 수정·삭제 정보 실시간 반영
- Spring SSE 기반 스트리밍
- Mattermost Chatbot 실시간 알림

#### <br>[SSE 알림]<br>
<img src="https://github.com/user-attachments/assets/9829ab57-10cb-47c3-bb64-aa395d2a3ed6" height="400" weight ="600"/>

#### <br>[Chatbot 긴급 알림]<br>
<img src="https://github.com/user-attachments/assets/08f05f8e-ec01-4b99-b27f-f9ce4a9929b1" height="400" weight ="600"/>

### 통합 검색 엔진 (Meilisearch)
- 메시지 + 파일명 + 파일 확장자 통합 검색
- 유사어, 오타 허용 검색 지원
- 변경/삭제 실시간 색인 반영

#### <br>[키워드 통합 검색]<br>
<img src="https://github.com/user-attachments/assets/491b7c77-7281-400c-8678-1788f0f5e9b5" height="400" weight ="600"/>

#### <br>[오타 검색]<br>
<img src="https://github.com/user-attachments/assets/657b37a7-643d-4552-ab09-a375c5354516" height="400" weight ="600"/>

#### <br>[유사어 검색]<br>
<img src="https://github.com/user-attachments/assets/719aeb30-d410-4b43-a7a2-2178a2ef3a36" height="400" weight ="600"/>

#### <br>[파일 검색]<br>
<img width="600" height="400" alt="6파일검색" src="https://github.com/user-attachments/assets/ebde7209-3df2-4b8a-9123-69483f8ee7f7" />

### Mattermost Plugin 연동
- Post Updated / Deleted Hook 기반 이벤트 처리
- Webhook → Backend → Search 자동 연동 구조
- 플러그인 persist 기반 지속성 유지


---

## 아키텍처
![Architecture](https://github.com/user-attachments/assets/9286307c-a641-4457-9f43-7e7258f6f500)

```
Mattermost
└─ Custom Plugin (Post Update/Delete Hook)
↓ Webhook
Spring Boot Server
├─ SSAFY SSO Login
├─ Webhook Event Processor
├─ AI Classification
├─ SSE Real-time Notification
├─ Chatbot Notification
└─ DB / FileInfo / Meilisearch Indexing
↓
Meilisearch
└─ 검색 API 제공
React Frontend (Vite)
└─ 실시간 스트림 UI / 검색 UI
```

---

## 인프라·배포·운영

인프라 담당 봉혜린은 컨테이너 기반 실행 환경과 Jenkins 배포 자동화를 구성하고, 운영 환경과 배포 절차를 [포팅 매뉴얼](exec/porting_manual.md)로 정리했습니다. 아래에서는 **저장소의 구현·문서로 확인되는 내용**과 **담당자 경험에 기반한 추가 확인 항목**을 구분합니다.

### Docker Compose 기반 운영 환경

포팅 매뉴얼에는 단일 AWS EC2(Ubuntu 24.04 LTS)에서 Docker Compose로 서비스를 운영하는 구성이 설명되어 있습니다. 애플리케이션과 데이터 저장소, 검색 엔진, 협업 플랫폼, 배포 도구를 함께 관리하는 구조입니다.

| 구성 요소 | 역할 및 확인 근거 |
| --- | --- |
| Spring Boot Backend | API·메시지 처리·AI 분류·알림. [Backend Dockerfile](backend/Dockerfile)에 Java 17 기반 다단계 이미지 빌드 정의 |
| Nginx / Frontend | [Frontend Dockerfile](frontend/Dockerfile)에서 React/Vite를 빌드하고 Nginx 이미지로 정적 파일 제공. HTTPS·API 프록시 구성은 포팅 매뉴얼에 설명 |
| MySQL | 관계형 데이터 저장소. 포팅 매뉴얼에 접속 환경 변수와 초기 설정 설명 |
| Redis | 포팅 매뉴얼에 운영 구성과 접속 환경 변수 설명 |
| MongoDB | 문서형 데이터 저장소. 포팅 매뉴얼에 접속 환경 변수 설명 |
| Meilisearch | 메시지·파일 통합 검색. 포팅 매뉴얼에 접속 설정 설명 |
| Jenkins | 이미지 빌드·레지스트리 업로드·원격 배포 자동화. [Jenkinsfile](Jenkinsfile)과 포팅 매뉴얼 참고 |
| Mattermost | 공지 수집·플러그인·챗봇 연동 대상. 포팅 매뉴얼에 별도 Compose 파일명과 연동 설정 설명 |

> **8컨테이너 구성 — 담당자 경험 기준 / 실물 설정 확인 필요**

> 담당자가 설명한 구성은 MySQL, Redis, MongoDB, Meilisearch, Backend, Nginx, Jenkins, Mattermost의 8개입니다. 위 표의 구성 요소는 소스·문서에서 확인되지만, 전체 Compose 파일은 현재 release 브랜치에 포함되어 있지 않아 실제 컨테이너 개수, 네트워크, 볼륨, 시작 의존성을 재현·검증할 수 없습니다. Nginx는 프론트엔드 이미지에도 포함되며, 매뉴얼에는 호스트 Nginx 설치도 설명되어 있어 별도 프록시와의 관계는 실제 설정 확인이 필요합니다.

### Jenkins CI/CD

[Jenkinsfile](Jenkinsfile)에 정의된 자동화 흐름은 다음과 같습니다.

1. 원본 GitLab 저장소의 `release` 브랜치 체크아웃
2. Gradle Backend 빌드 및 React/Vite Frontend Docker 이미지 빌드
3. Backend Docker 이미지 빌드, 프론트·백엔드 이미지에 빌드 번호와 `latest` 태그 부여
4. Jenkins Credentials를 사용해 Docker Hub에 이미지 업로드
5. SSH로 배포 서버에 접속해 Compose 서비스 갱신
6. 로컬·서버 이미지 정리 및 파이프라인 성공·실패 기록

빌드 번호 태그로 배포 이미지 버전을 식별할 수 있습니다. 다만 매뉴얼의 Compose 예시는 `latest`를 사용하며, 실제 Compose 파일이 없으므로 서버가 빌드 번호 태그를 사용하는지는 확인되지 않습니다. Backend의 Gradle 빌드 단계는 `-x test`로 테스트를 생략하므로 이 파이프라인을 자동 테스트까지 포함한 검증 절차로 해석하지 않습니다.

### Nginx와 실시간 알림 운영

[포팅 매뉴얼](exec/porting_manual.md)은 HTTPS, API 라우팅, React SPA 정적 파일 제공 외에 SSE 경로의 `proxy_buffering off`, `proxy_cache off`, `proxy_read_timeout 86400s`를 설명합니다. SSE 이벤트가 프록시 버퍼에 머무르지 않도록 하고 긴 연결을 수용하려는 구성입니다. 전체 `nginx.conf`는 저장소에 없으므로 이는 **매뉴얼에 기록된 설정**입니다.

코드에서는 [AsyncPostProcessor](backend/src/main/java/com/A105/prham/webhook/service/AsyncPostProcessor.java)의 `@Async` 이벤트 처리와 일부 알림의 `CompletableFuture.runAsync`를 확인할 수 있습니다. [SsePostService](backend/src/main/java/com/A105/prham/sse/service/SsePostService.java)는 연결 종료·타임아웃·오류 시 구독 정보를 정리합니다. 비동기 처리와 SSE 구현은 확인되지만, 이것만으로 특정 응답 시간이나 개선 원인을 입증하지는 않습니다.

### 담당자 운영 경험 및 성과 — 추가 확인 필요

다음은 담당자가 설명한 프로젝트 운영 경험입니다. 현재 release 브랜치의 설정·로그·측정 자료만으로 검증되지 않아 구현 사실과 분리해 기록합니다.

| 항목 | 담당자 설명 | 현재 저장소의 확인 범위 / 추가로 필요한 근거 |
| --- | --- | --- |
| Docker Compose 구성 | 8컨테이너 환경 구성 | 구성 요소와 Compose 사용은 문서에 있음. 전체 Compose 파일과 당시 컨테이너 목록 필요 |
| 배포 자동화 | 수동 배포 약 7분에서 Jenkins 자동 배포로 전환 | 자동화 파이프라인은 확인됨. 7분의 측정 기록과 자동 배포 소요 시간은 없음 |
| Nginx Blue-Green 배포·무중단 운영 | 기존 배포 시 약 2~3분 중단을 개선 | 현재 Jenkinsfile은 `docker compose down` → `pull` → `up -d` 순서여서 서비스 중단이 발생할 수 있음. Blue/Green 환경, 헬스 검증, Nginx upstream 전환·롤백 구현은 확인되지 않음 |
| 응답 시간 개선 | 약 2~3초 → 1초 미만 | 대상 API·기능, 측정 시작/종료 지점, 요청 수·동시 사용자 수, 평균/p95 등 통계와 전후 로그가 없어 검증되지 않음 |

Blue-Green 경험을 현재 코드와 연결하려면 실제 사용한 Compose·Nginx 설정과 전환 스크립트의 버전 및 운영 시점을 함께 제시해야 합니다. 특히 SSE 연결 정보가 애플리케이션 메모리에 있으므로, **신규 요청의 무중단 전환과 기존 SSE 연결의 유지 여부는 별도로 검증**해야 합니다. 위 경험은 현재 Jenkinsfile이 무중단 배포를 수행한다는 의미가 아닙니다.

### 배포 문서와 재현 범위

서버 구성, 환경 변수, 배포 절차는 [포팅 매뉴얼](exec/porting_manual.md)을 참고하세요. 이 문서는 운영 당시의 설명이며, 현재 저장소만으로 바로 실행할 수 있는 완결된 배포 패키지는 아닙니다.

- 매뉴얼에 열거된 `docker-compose-infra.yml`, `docker-compose-jenkins.yml`, `docker-compose-mattermost.yml`, `docker-compose.yml`, `nginx.conf`, `Dockerfile.jenkins`의 실물 파일은 release 브랜치에 없습니다.
- 서버 환경 변수, Jenkins Credentials·도구·플러그인, 네트워크·볼륨·포트 설정은 실제 운영 환경 확인이 필요합니다.
- Blue-Green 전환·롤백 명령은 저장소에서 확인할 수 없어 실행 예시를 추가하지 않았습니다. 기존 배포 명령은 서비스 중지를 포함하므로 무중단 배포 절차로 사용하지 않습니다.

---

## ERD

![ERD](https://github.com/user-attachments/assets/99e4325a-6c65-4b77-a59c-ce263076d60b)


---

## 기술 스택


| Category | Technologies |
|----------|-------------|
| **Frontend** | ![React](https://img.shields.io/badge/-React-20232A?logo=react&logoColor=61DAFB) ![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?logo=typescript&logoColor=white) ![Zustand](https://img.shields.io/badge/-Zustand-443E38?logo=zustand&logoColor=white) |
| **Backend** | ![Spring Boot](https://img.shields.io/badge/-Spring_Boot-6DB33F?logo=springboot&logoColor=white) ![Java](https://img.shields.io/badge/-Java-007396?logo=openjdk&logoColor=white) ![MySQL](https://img.shields.io/badge/-MySQL-4479A1?logo=mysql&logoColor=white) ![MongoDB](https://img.shields.io/badge/-MongoDB-47A248?logo=mongodb&logoColor=white) ![MeiliSearch](https://img.shields.io/badge/-MeiliSearch-4E75F6?logo=meilisearch&logoColor=white) |
| **AI** | ![OpenAI](https://img.shields.io/badge/-OpenAI-412991?logo=openai&logoColor=white) |
| **DevOps** | ![Docker](https://img.shields.io/badge/-Docker-2496ED?logo=docker&logoColor=white) ![Jenkins](https://img.shields.io/badge/-Jenkins-D24939?logo=jenkins&logoColor=white) |
| **Collaboration** | ![Git](https://img.shields.io/badge/-Git-F05032?logo=git&logoColor=white) ![GitLab](https://img.shields.io/badge/-GitLab-FC6D26?logo=gitlab&logoColor=white) ![Jira](https://img.shields.io/badge/-Jira-0052CC?logo=jira&logoColor=white) ![Notion](https://img.shields.io/badge/-Notion-000000?logo=notion&logoColor=white) |

---


## API 명세

- [API 문서 바로가기](https://gilded-angle-55e.notion.site/API-27d67182e119814a9183c2dfec85c698?source=copy_link)

---

## 팀 구성
2025.10.10~2025.11.20 <br>

| 이름 | 역할 |
|------|------|
| 이현석 | Frontend |
| 조영우 | Frontend |
| 김은재 | Backend |
| 이예린 | Backend |
| 석예은 | Backend |
| 봉혜린 | Infra — 컨테이너 실행 환경 구성, Jenkins CI/CD, 배포·운영 문서화 (상세 및 확인 범위는 인프라·배포·운영 참고) |

---
