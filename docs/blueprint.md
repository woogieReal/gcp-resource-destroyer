# gcp-resource-destroyer

> **Cloud Tasks 기반으로 예약된 시간에 GitHub 내 테라폼 리소스를 자동 삭제하는 서버리스 폭파 봇**

임시 테스트 인프라 생성 후 끄는 것을 잊어버려 발생하는 비용 누수를 방지하기 위해, 지정된 TTL(Time-To-Live) 경과 시 원격 레포지토리의 테라폼 리소스를 자동으로 정리하는 이벤트 기반 자동화 파이프라인 명세서입니다.

---

## 1. System Architecture

```text
[Developer / CI]
      │
      │ 1. POST /reserve (TTL, Target Repo URL, Working Dir)
      ▼
┌──────────────────────────────────────────────┐
│ Cloud Run (Destroyer API)                    │
│ Service Account: destroyer-app-sa            │
└──────────────────────────────────────────────┘
      │
      │ 2. Task 생성 (scheduleTime = Now + TTL)
      │    * OIDC Token 인증 헤더 포함
      ▼
┌──────────────────────────────────────────────┐
│ Cloud Tasks (destroy-queue)                  │
└──────────────────────────────────────────────┘
      │
      │ 3. 예약 시간 도래 시 HTTP POST /internal/destroy 트리거
      │    (OIDC Token 검증을 통한 보안 호출)
      ▼
┌──────────────────────────────────────────────┐
│ Cloud Run (Destroyer Runner Container)       │
│                                              │
│  - Git Clone (타겟 공개 레포지토리)          │
│  - GOOGLE_IMPERSONATE_SERVICE_ACCOUNT 주입   │
│  - terraform init && terraform destroy       │
└──────────────────────────────────────────────┘
      │
      │ 4. Resource Destroy (사칭된 권한으로 수행)
      ▼
┌──────────────────────────────────────────────┐
│ GCP Target Resources (woogie-sandbox-gcp)    │
└──────────────────────────────────────────────┘

```

## 2. Core Design Principles

* **Zero Key File (키 파일 없음):**
* 서비스 계정의 `key.json`을 절대 사용하거나 보관하지 않습니다.
* 실행 주체인 Cloud Run 서비스 계정(`destroyer-app-sa`)이 프로젝트 내 마스터 계정(`master-destroyer-sa`)을 사칭(Impersonation)하여 단기 인증 토큰만으로 인프라를 삭제합니다.


* **Internal Only API (내부 보안 통신):**
* 실제 삭제를 수행하는 `/internal/destroy` 엔드포인트는 퍼블릭 망에서 직접 호출할 수 없습니다.
* 오직 OIDC 토큰을 발급받은 Cloud Tasks 큐만이 트리거할 수 있도록 IAM 기반 접근 제어(Require Auth)를 적용합니다.


* **Single Project Optimization (단일 샌드박스 최적화):**
* 대상 프로젝트는 항상 `woogie-sandbox-gcp`로 고정됩니다.
* 최초 1회의 IAM 권한 바인딩 이후, 개별 레포지토리 배포 시에는 어떠한 인증 관련 코드나 하드코딩도 필요하지 않습니다.



---

## 3. API Specifications

### 3.1 폭파 예약 API (External)

클라이언트나 CI/CD 파이프라인에서 인프라 배포 성공 직후 호출하여 폭파 일정을 예약합니다.

* **Endpoint:** `POST /reserve`
* **Content-Type:** `application/json`

**Request Body**

```json
{
  "github_repo": "https://github.com/woogieReal/sandbox-temp-infra",
  "working_dir": "terraform",
  "ttl_hours": 3
}

```

**Response (201 Created)**

```json
{
  "message": "Destroy task scheduled",
  "scheduledTime": 1718293847
}

```

### 3.2 폭파 실행 트리거 (Internal)

Cloud Tasks가 예약된 시간에 OIDC 인증 토큰을 동반하여 호출하는 내부 전용 엔드포인트입니다.

* **Endpoint:** `POST /internal/destroy`
* **Security:** IAM OIDC Token Required

---

## 4. IAM & Security Configuration

이 시스템이 동작하기 위해 `woogie-sandbox-gcp` 프로젝트 내에 1회성으로 구성해야 하는 보안 설정입니다.

| 서비스 계정 | 목적 | 부여된 IAM 역할 |
| --- | --- | --- |
| `destroyer-app-sa` | Cloud Run 런타임 및 Task 생성 주체 | `roles/cloudtasks.enqueuer`<br>

<br>`roles/run.invoker` |
| `master-destroyer-sa` | 타겟 리소스를 파괴하는 마스터 계정 | `roles/editor` (프로젝트 전역) |

**권한 사칭(Impersonation) 바인딩:**
`destroyer-app-sa`가 `master-destroyer-sa`의 토큰을 획득할 수 있도록 `roles/iam.serviceAccountTokenCreator` 권한 다리를 연결합니다.

---

## 5. Directory Structure

```text
gcp-resource-destroyer/
├── src/
│   ├── index.ts                # Express 라우팅 및 전체 API 엔드포인트
│   ├── config.ts               # 환경 변수 (프로젝트 ID, 리전 등)
├── terraform/
│   ├── main.tf                 # Cloud Tasks 큐, Service Accounts, IAM 바인딩 인프라 정의
├── Dockerfile                  # Node.js + Git + Terraform CLI 포함 커스텀 이미지
├── package.json
├── .gitignore
└── README.md                   # 본 설계 문서

```