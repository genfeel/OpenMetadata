# OpenMetadata 아키텍처 분석

> **버전**: 1.12.0-SNAPSHOT
> **분석일**: 2026-03-08
> **저장소**: <https://github.com/open-metadata/OpenMetadata>

---

## 목차

1. [기술 스택](#1-기술-스택)
2. [모듈 의존성 구조](#2-모듈-의존성-구조)
3. [전체 아키텍처 다이어그램](#3-전체-아키텍처-다이어그램)
4. [openmetadata-spec (스키마 모듈)](#4-openmetadata-spec-스키마-모듈)
5. [openmetadata-service (Java 백엔드)](#5-openmetadata-service-java-백엔드)
6. [openmetadata-ui (React 프론트엔드)](#6-openmetadata-ui-react-프론트엔드)
7. [ingestion (Python 인제스천 프레임워크)](#7-ingestion-python-인제스천-프레임워크)

---

## 1. 기술 스택

| 계층 | 기술 | 버전 |
| ---- | ---- | ---- |
| 백엔드 | Java + Dropwizard | Java 21, Dropwizard 5.0 |
| 프론트엔드 | React + TypeScript | React 18.2, Vite 7.1 |
| UI 라이브러리 | Ant Design에서 MUI로 전환 중 | AntD 4.24, MUI 7.3.1 |
| UI 코어 | React Aria + Tailwind v4 | 신규 공유 컴포넌트 라이브러리 |
| 상태 관리 | Zustand | 4.5 |
| 인제스천 | Python + Pydantic | Python 3.9-3.12, Pydantic 2.x |
| 데이터베이스 | MySQL / PostgreSQL | Flyway 마이그레이션 |
| 검색 엔진 | Elasticsearch / OpenSearch | ES 7.17+ / OS 2.6+ |
| 직렬화 | Jackson | 2.17.2 |
| HTTP | Jersey (JAX-RS) | 3.1.11 |
| 워크플로우 | Apache Airflow | 3.1.5 (선택사항) |

---

## 2. 모듈 의존성 구조

### Maven 모듈 (빌드 순서)

```text
openmetadata-spec (JSON Schema에서 Java 모델 생성)
       │
       ├──→ openmetadata-sdk (API 클라이언트)
       │          │
       │          └──→ openmetadata-integration-tests
       │
       └──→ openmetadata-service (핵심 백엔드, 의존성 168개)
                  │
                  ├──→ openmetadata-k8s-operator
                  ├──→ openmetadata-mcp (MCP 서버)
                  └──→ openmetadata-dist (최종 배포 패키지)
                            ├── openmetadata-service
                            ├── openmetadata-mcp
                            └── openmetadata-ui

openmetadata-ui-core-components (독립 UI 라이브러리)
       └──→ openmetadata-ui (프론트엔드)

common (공통 유틸리티, 독립)
openmetadata-shaded-deps (쉐이딩된 의존성, 독립)
openmetadata-clients (API 클라이언트 생성, 독립)
```

### 모듈별 역할

| 모듈 | 의존성 수 | 역할 |
| ---- | --------- | ---- |
| `openmetadata-spec` | 14 | JSON Schema 정의, 코드 생성의 원천 |
| `openmetadata-sdk` | 11 | Java API 클라이언트, spec에 의존 |
| `common` | 7 | 공통 유틸리티, 코드 생성 어노테이터 |
| `openmetadata-shaded-deps` | 0 | 의존성 쉐이딩 (충돌 해결) |
| `openmetadata-service` | 168 | 핵심 백엔드: REST API, 엔티티, 이벤트, 인증 |
| `openmetadata-k8s-operator` | 16 | 쿠버네티스 오퍼레이터, service에 의존 |
| `openmetadata-mcp` | 15 | MCP 서버, service에 의존 |
| `openmetadata-ui-core-components` | 0 (npm) | 공유 React 컴포넌트 라이브러리 |
| `openmetadata-ui` | 0 (npm) | React 프론트엔드 애플리케이션 |
| `openmetadata-dist` | 4 | 최종 패키징 (service + mcp + ui) |
| `openmetadata-clients` | 0 | 생성된 API 클라이언트 |
| `openmetadata-integration-tests` | 23 | 통합 테스트 스위트 |

---

## 3. 전체 아키텍처 다이어그램

```text
┌─────────────────────────────────────────────────────────┐
│                   openmetadata-spec                      │
│          (786개 JSON Schema - 단일 진실의 원천)            │
│         Java ← jsonschema2pojo | TS ← json2ts           │
│              Python ← datamodel-code-generator           │
└───────────────┬──────────────┬──────────────┬───────────┘
                │              │              │
    ┌───────────▼──┐   ┌──────▼──────┐  ┌────▼─────────┐
    │  백엔드      │   │  프론트엔드  │  │  인제스천     │
    │  (Java/DW)   │   │  (React)    │  │  (Python)    │
    │              │   │             │  │              │
    │ REST API ────┼──►│ Axios/REST  │  │ 75+ 소스     │
    │ LMAX 이벤트  │   │ Zustand     │  │ 토폴로지 실행 │
    │ JDBI/SQL     │   │ 지연 로딩    │  │ Either<T>    │
    │ SpEL 인가    │   │ MUI+AntD    │  │ ServiceSpec  │
    └──────┬───────┘   └─────────────┘  └──────┬───────┘
           │                                    │
    ┌──────▼────────────────────────────────────▼──────┐
    │          MySQL / PostgreSQL + ES/OpenSearch       │
    └──────────────────────────────────────────────────┘
```

---

## 4. openmetadata-spec (스키마 모듈)

### 역할

전체 시스템의 **단일 진실의 원천(Single Source of Truth)**. 786개의 JSON Schema 파일이 Java, TypeScript, Python 모델을 생성한다.

### 디렉토리 구조

```text
openmetadata-spec/
├── src/main/
│   ├── resources/json/schema/           # 786개 JSON Schema 파일
│   │   ├── entity/                      # 핵심 메타데이터 엔티티
│   │   ├── type/                        # 공유 타입 정의 및 열거형
│   │   ├── api/                         # API 요청/응답 스키마
│   │   ├── security/                    # 인증 및 자격증명 설정
│   │   ├── metadataIngestion/           # 인제스천 파이프라인 설정
│   │   ├── governance/                  # 워크플로우 및 거버넌스 스키마
│   │   ├── events/                      # 이벤트 스키마
│   │   ├── configuration/              # 시스템 설정 스키마
│   │   ├── elasticsearch/              # ES 인덱싱 설정 (다국어)
│   │   └── rdf/                        # RDF 온톨로지 정의
│   ├── java/org/openmetadata/          # 핵심 인터페이스 및 유틸리티
│   │   ├── schema/                     # EntityInterface, ServiceEntityInterface 등
│   │   ├── search/                     # IndexMapping, IndexMappingLoader
│   │   ├── annotations/               # 코드 생성 어노테이션
│   │   └── utils/                      # JSON 유틸리티, 버전 관리
│   └── antlr4/                         # FQN 파싱을 위한 ANTLR 문법
└── pom.xml
```

### 코드 생성 파이프라인

| 언어 | 도구 | 명령어 | 출력 위치 |
| ---- | ---- | ------ | --------- |
| **Java** | jsonschema2pojo (Maven) | `mvn clean install` | `org.openmetadata.schema.*` |
| **Python** | datamodel-code-generator | `make generate` | `ingestion/.../generated/schema/` |
| **TypeScript** | json2ts | `./json2ts-generate-all.sh` | `ui/src/generated/` |
| **ANTLR** | ANTLR4 | `make py_antlr` / `make js_antlr` | FQN 파싱 (Python & JS) |

**커스터마이징**: OpenMetadataAnnotator(ExposedAnnotator, MaskedAnnotator, PasswordAnnotator, DeprecatedAnnotator로 구성)가 Java 코드 생성 시 커스텀 어노테이션을 추가한다.

### 엔티티 유형 (총 786개 스키마)

| 분류 | 주요 엔티티 |
| ---- | ----------- |
| **데이터** (`entity/data/`) | Table, Database, DatabaseSchema, Dashboard, Chart, Pipeline, Topic, SearchIndex, Container, APIEndpoint, Glossary, GlossaryTerm, Query, StoredProcedure, Metric |
| **조직** (`entity/teams/`) | User, Team (Organization → BusinessUnit → Division → Department → Group) |
| **서비스** (`entity/services/`) | DatabaseService, DashboardService, PipelineService, MessagingService, StorageService, SearchService, APIService, MLModelService, LLMService + 75개 이상의 연결 유형 |
| **거버넌스** (`entity/governance/`) | Workflow, Policy, AccessControl, Classification, Domain, DataProduct, DataContract, TestCase |
| **지원** | Applications, Bots, Events, Feed, DocStore, AIGovernancePolicy |

### 스키마 설계 패턴

**A. 엔티티 기본 구조** — 모든 엔티티는 `EntityInterface`를 구현한다:

```json
{
  "javaType": "org.openmetadata.schema.entity.data.Table",
  "javaInterfaces": ["org.openmetadata.schema.EntityInterface"],
  "properties": {
    "id": { "$ref": "../../type/basic.json#/definitions/uuid" },
    "name": { "$ref": "../../type/basic.json#/definitions/entityName" },
    "fullyQualifiedName": { "$ref": "../../type/basic.json#/definitions/fullyQualifiedEntityName" },
    "version": { "$ref": "../../type/entityHistory.json#/definitions/entityVersion" }
  }
}
```

**B. `$ref`를 활용한 조합(Composition)** — 파일 간 참조로 타입을 재사용한다:

- 로컬 참조: `"$ref": "#/definitions/customType"`
- 파일 간 참조: `"$ref": "../../type/basic.json#/definitions/uuid"`

**C. `oneOf`를 활용한 다형성** — 연결 설정에서 다중 인증 방식을 지원한다:

```json
{
  "authType": {
    "oneOf": [
      { "$ref": "./common/basicAuth.json" },
      { "$ref": "./common/iamAuthConfig.json" },
      { "$ref": "./common/azureConfig.json" }
    ]
  }
}
```

**D. EntityReference** — 전체 객체를 포함하지 않고 엔티티를 연결하는 소프트 참조:

```json
{ "id": "uuid", "type": "entity-type-string", "name": "string", "fullyQualifiedName": "string" }
```

**E. 커스텀 어노테이션** — 코드 생성 시 자격증명 처리를 위한 `mask`, `password` 플래그.

### 핵심 통찰

1. **스키마 우선 설계**: 스키마가 정규 소스이며 코드는 항상 생성된다
2. **계층화된 타입 시스템**: 기본 타입 → 도메인 타입 → 엔티티 타입 → API 타입
3. **다중 타깃 생성**: 단일 스키마에서 Java + Python + TypeScript + ANTLR 파서를 생성한다
4. **인터페이스 기반**: 모든 엔티티가 EntityInterface를 구현하여 일관된 동작을 보장한다

---

## 5. openmetadata-service (Java 백엔드)

### 진입점

`OpenMetadataApplication.java` — Dropwizard 애플리케이션

### 패키지 구조

```text
openmetadata-service/
├── OpenMetadataApplication.java          # 메인 진입점
├── Entity.java                            # 중앙 엔티티 레지스트리
├── resources/                             # REST API 엔드포인트 (55개 이상의 패키지)
│   ├── EntityResource.java                # 모든 엔티티 리소스의 기본 클래스
│   ├── CollectionRegistry.java            # REST 컬렉션 자동 발견
│   └── [엔티티별 리소스]
├── jdbi3/                                 # 데이터 접근 계층 (109개 클래스)
│   ├── EntityRepository.java              # 기본 리포지토리 클래스
│   ├── EntityDAO.java                     # JDBI 기반 DAO 인터페이스
│   └── [엔티티별 리포지토리]
├── security/                              # 인증 및 인가
│   ├── Authorizer.java / DefaultAuthorizer.java
│   ├── auth/                              # 인증 핸들러
│   └── policyevaluator/                   # SpEL 기반 정책 평가
├── events/                                # 이벤트 기반 아키텍처
│   ├── EventPubSub.java                   # LMAX Disruptor 이벤트 버스
│   ├── AbstractEventPublisher.java
│   └── [특정 핸들러]
├── search/                                # Elasticsearch/OpenSearch
├── apps/                                  # 애플리케이션 플러그인
├── audit/                                 # 감사 로깅
├── cache/                                 # 캐싱 계층
├── jobs/                                  # 백그라운드 작업
├── migration/                             # 데이터 마이그레이션
├── monitoring/                            # 모니터링
└── notifications/                         # 알림 시스템
```

### 3계층 데이터 접근

```text
┌─────────────────────────────────────────────────┐
│  REST 계층: EntityResource<T, K>                │
│  - 55개 이상의 리소스를 @Collection으로 자동 발견  │
│  - CRUD + 인가 체크                              │
│  - CSV 임포트/익스포트, 대량 작업                 │
├─────────────────────────────────────────────────┤
│  리포지토리 계층: EntityRepository<T>             │
│  - 변경 추적 및 버전 관리                         │
│  - ETag 기반 낙관적 동시성 제어                   │
│  - 소프트 삭제 및 복원                            │
│  - ChangeEvent를 EventPubSub에 발행              │
├─────────────────────────────────────────────────┤
│  데이터 접근 계층: EntityDAO<T> (JDBI)            │
│  - @ConnectionAwareSqlQuery (MySQL/PostgreSQL)   │
│  - 배치 작업 및 트랜잭션                          │
└─────────────────────────────────────────────────┘
```

**엔티티 레지스트리** (`Entity.java`):

- 정적 매핑: `entityType (String) → EntityRepository`
- 중앙 접근: `Entity.getEntityRepository("database")` → `DatabaseRepository`

### 이벤트 시스템 (LMAX Disruptor)

```text
Repository.save()
  → EventPubSub.publish(ChangeEvent)
  → [LMAX Disruptor RingBuffer (1024 슬롯)]
  ↓
  다수의 비동기 핸들러:
    ├─ AuditLogEventPublisher     (감사 추적)
    ├─ EventMonitorPublisher      (메트릭 및 모니터링)
    ├─ ChangeEventRepository      (이벤트 이력)
    ├─ SearchIndexPublisher       (ES/OpenSearch 인덱싱)
    ├─ WebSocketPublisher         (실시간 알림)
    └─ [커스텀 핸들러]
```

### 인증 / 인가

**인증** (플러그인 방식):

- JWT / OAuth2 / SAML / OIDC / LDAP
- 구현체: BasicAuthenticator, LdapAuthenticator, NoopAuthenticator

**인가** (정책 기반):

```text
DefaultAuthorizer implements Authorizer
  └─ PolicyEvaluator (SpEL 규칙 엔진)
     ├─ SubjectContext (사용자 + 역할 + 속성)
     ├─ ResourceContext (대상 리소스 정보)
     └─ OperationContext (수행 중인 작업)
```

권한 매트릭스: `리소스 x 작업` (예: `Database.CREATE`, `User.EDIT_OWNER`)

### 초기화 순서

```text
OpenMetadataApplication.run()
  1. 핵심 설정: JDBI, Config, Fernet 암호화
  2. 엔티티 초기화: collectionDAO, 리포지토리
  3. 검색 인프라: SearchRepository 설정
  4. 보안: Authorizer 및 Authenticator 등록
  5. 리소스: CollectionRegistry가 REST 엔드포인트를 스캔 및 등록
  6. 이벤트: EventPubSub 시작 + 핸들러 등록
  7. 웹소켓: Feed 서블릿 및 소켓 설정
  8. 라이프사이클: 작업 핸들러, 백그라운드 워커
```

### 설계 패턴

| 패턴 | 위치 | 목적 |
| ---- | ---- | ---- |
| **리포지토리** | `jdbi3/` | JDBI SQL 위의 제네릭 CRUD 추상화 |
| **팩토리** | Auth, Search, EventMonitor | 설정 기반 구현체 생성 |
| **전략** | `Authorizer` 인터페이스 | 플러그 가능한 인가 전략 |
| **템플릿 메서드** | `EntityRepository` | 후크가 있는 CRUD 워크플로우 |
| **데코레이터** | `MessageDecorator`, `EntityMasker` | 데이터 포맷팅/마스킹 |
| **옵저버/Pub-Sub** | `EventPubSub` + Disruptor | 변경과 처리를 분리 |
| **플러그인** | `@Repository`, `@Collection` | 자동 발견 및 등록 |
| **레지스트리** | `CollectionRegistry`, `Entity.java` | 타입 안전한 객체 조회 |

### 주요 특징

- **상속보다 조합**: EntityReference 소프트 링크 (JPA 상속 없음)
- **FQN (정규화된 이름)**: 고유한 계층적 이름 (예: `db.schema.table`)
- **버전 관리**: 모든 엔티티가 변경 설명과 함께 버전 이력을 유지
- **소프트 삭제**: 물리적 삭제가 아닌 삭제 표시 (복원 지원)
- **ETag 동시성**: ETag를 통한 낙관적 잠금
- **연결 인식 SQL**: 데이터베이스 유형에 따라 쿼리 구현을 전환

---

## 6. openmetadata-ui (React 프론트엔드)

### 빌드 및 설정

- **빌드 도구**: Vite 7.1.11 + TypeScript 5.0
- **코드 분할**: 수동 청크 (React vendor, Ant Design vendor, Editor vendor, Chart vendor)
- **압축**: 프로덕션에서 Gzip + Brotli
- **경로 별칭**: `@/ = src/`

### 컴포넌트 구성

**명명 규칙**:

```text
ComponentName.component.tsx   # 컴포넌트 구현
ComponentName.interface.ts    # Props 및 타입 정의
ComponentName.utils.ts        # 헬퍼 함수
ComponentName.test.tsx         # 단위 테스트
component-name.less            # 스타일 (BEM 명명)
```

**디렉토리 구조**:

```text
src/
├── components/
│   ├── common/              # 공유 UI (40개 이상의 컴포넌트)
│   │   ├── atoms/           # 최소 단위 UI 블록 (MUI 전환용)
│   │   ├── Form/            # 폼 컴포넌트
│   │   └── Loader/          # 로딩 인디케이터
│   ├── [도메인별]            # Database, Entity, Explore, Dashboard 등
│   ├── AppRouter/           # HOC가 포함된 라우팅
│   ├── AppBar/              # 내비게이션
│   └── Settings/            # 설정 UI
├── pages/                   # 라우트 레벨 페이지 컴포넌트
├── rest/                    # API 계층 (77개 파일)
├── generated/               # 자동 생성된 TypeScript 타입
├── hooks/                   # 커스텀 훅
├── context/                 # React Context 프로바이더
├── utils/                   # 유틸리티 함수
├── locale/                  # 다국어(i18n) 번역
├── constants/               # 애플리케이션 상수
├── enums/                   # TypeScript 열거형
└── styles/                  # 전역 스타일
```

### 라우팅 아키텍처

```text
AppRouter (인증 확인)
  ├─ AuthenticatedAppRouter (보호됨, 지연 로딩)
  │   ├─ EntityRouter         # 엔티티 상세 페이지
  │   ├─ SettingsRouter       # 설정 페이지
  │   ├─ DomainRouter         # 도메인 관리
  │   ├─ GlossaryRouter       # 용어집 관리
  │   ├─ ClassificationRouter # 분류 관리
  │   └─ AdminProtectedRoute  # 관리자 전용 래퍼
  └─ UnAuthenticatedAppRouter (공개)
      └─ 로그인, 회원가입 페이지
```

- **지연 로딩**: `React.lazy()` + `withSuspenseFallback` HOC
- **라우트 상수**: `src/constants/constants.ts`에 정의

### 상태 관리 (3계층)

| 범위 | 도구 | 예시 |
| ---- | ---- | ---- |
| **전역** | Zustand | `useApplicationStore` (사용자, 인증, 설정, 테마) |
| **기능별** | React Context | `PermissionProvider`, `WebSocketProvider`, `LineageProvider` |
| **로컬** | `useState` | UI 상태 (모달, 폼 입력, 필터) |

**프로바이더 계층** (App.tsx):

```text
BrowserRouter
  └─ I18nextProvider
     └─ ThemeProvider
        └─ SnackbarProvider
           └─ AuthProvider
              └─ PermissionProvider
                 └─ WebSocketProvider
                    └─ AppRouter
```

### API 계층

**기본 클라이언트** (`src/rest/index.ts`):

```typescript
const axiosClient = axios.create({
  baseURL: `${getBasePath()}/api/v1`,
  paramsSerializer: (params) => Qs.stringify(params, { arrayFormat: 'comma' }),
});
```

**패턴** (`src/rest/`에 77개 API 파일):

```typescript
export const getApiEndPoints = async (params: GetApiEndPointsType) => {
  const response = await APIClient.get<PagingResponse<APIEndpoint[]>>(
    `/apiEndpoints`, { params }
  );
  return response.data;
};
```

- 모든 응답은 `/generated/`의 생성된 모델로 타입이 지정됨
- 페이지네이션: `PagingResponse<T[]>`, `PagingWithoutTotal`
- 에러 처리: 컴포넌트에서 try-catch → `showErrorToast()` / `showSuccessToast()`

### UI 전환 현황

**현재 상태**: Ant Design에서 Material-UI로 전환 중

- **레거시**: Ant Design 4.24 + LESS 스타일
- **신규**: MUI 7.3.1 + `ui-core-components` (React Aria + Tailwind v4)
- **테마**: MUI 테마는 `openmetadata-ui-core-components`에서 정의
- **전략**: 신규 컴포넌트는 MUI 사용, 기존 컴포넌트는 점진적으로 전환

### 주요 패턴

- **함수형 컴포넌트만 사용**: 클래스 컴포넌트 없음
- **다국어(i18n)**: `useTranslation()` 훅 — UI에 문자열 리터럴 사용 금지
- **성능 최적화**: `useCallback`, `useMemo`, `useShallow` (Zustand)
- **폼**: 동적 스키마 기반 폼에 RJSF, 커스텀 폼에 React Hook Form
- **인증**: 다중 전략 (OAuth2, SAML, OIDC, Azure AD, Okta)
- **테스트**: Jest + React Testing Library (단위), Playwright (E2E)

---

## 7. ingestion (Python 인제스천 프레임워크)

### 패키지 구조

```text
ingestion/src/metadata/ingestion/
├── api/                    # 핵심 워크플로우 추상화 및 실행 엔진
├── connections/            # 연결 관리 (빌더, 인증, 세션)
├── ometa/                  # OpenMetadata API 클라이언트 (믹스인 포함)
├── source/                 # 데이터 소스 커넥터
│   ├── database/           # 70개 이상의 데이터베이스 커넥터
│   ├── dashboard/          # 대시보드 도구 (Tableau, Superset, Looker 등)
│   ├── pipeline/           # 오케스트레이션 (Airflow, Dagster, Fivetran 등)
│   ├── messaging/          # 메시지 브로커 (Kafka 등)
│   ├── mlmodel/            # ML 모델 레지스트리 (MLflow 등)
│   ├── storage/            # 오브젝트 스토리지 (S3, GCS, Azure 등)
│   └── metadata/           # 메타데이터 소스 (dbt 등)
├── processor/              # 데이터 변환/필터링
├── sink/                   # 출력 (metadata_rest.py → OpenMetadata API)
├── stage/                  # 대량 작업을 위한 스테이징
├── models/                 # Pydantic 데이터 모델
└── bulksink/               # 대량 작업 싱크
```

### 파이프라인 패턴: Source → Processor → Sink

**단계(Step) 유형**:

| 단계 | 기본 클래스 | 메서드 | 역할 |
| ---- | ----------- | ------ | ---- |
| **Source** | `IterStep` | `_iter()` | 데이터 소스에서 엔티티를 생성(yield) |
| **Processor** | `ReturnStep` | `_run(record)` | 엔티티를 변환/필터링 |
| **Sink** | `ReturnStep` | `_run(record)` | 엔티티를 목적지로 전송 |
| **Stage** | `StageStep` | `_run(record)` | 파일로 데이터를 스테이징 |
| **BulkSink** | `BulkStep` | `run()` | 스테이징된 데이터에 대한 대량 작업 |

**에러 처리** — 함수형 `Either[T]` 타입:

```python
class Either(BaseModel, Generic[T]):
    left: Optional[StackTraceError]   # 에러 케이스
    right: Optional[T]                 # 성공 케이스
```

### 토폴로지 기반 실행

```text
get_services() → [서비스 목록]
  └→ get_databases() → [데이터베이스 목록]
     └→ get_database_schemas() → [스키마 목록]
        └→ get_tables() → [테이블 목록]
           ├→ get_columns() → [컬럼 목록]
           └→ get_table_constraints() → [제약 조건 목록]
```

각 토폴로지 노드는 **producer** 메서드와 처리를 위한 **stage**를 가진다.

### 커넥터 플러그인 아키텍처

**표준 커넥터 구조**:

```text
source/database/{connector_name}/
├── metadata.py      # CommonDbSourceService를 상속
├── connection.py    # BaseConnection<Config, Client>를 상속
├── service_spec.py  # 선언적 등록 매니페스트
├── lineage.py       # (선택) 리니지 추출
├── usage.py         # (선택) 사용량 추적
├── queries.py       # 커넥터별 SQL
└── models.py        # 커넥터별 모델
```

**서비스 명세(ServiceSpec)** (선언적 등록):

```python
from metadata.utils.service_spec.default import DefaultDatabaseSpec

ServiceSpec = DefaultDatabaseSpec(
    metadata_source_class=MysqlSource,
    lineage_source_class=MysqlLineageSource,
    usage_source_class=MysqlUsageSource,
    connection_class=MySQLConnection,
)
```

### 연결 관리

**BaseConnection** — 제네릭, 타입 안전한 추상 클래스:

```python
class BaseConnection(ABC, Generic[S, C]):
    service_connection: S   # Pydantic 설정 모델
    _client: Optional[C]    # 지연 로딩되는 클라이언트

    @property
    def client(self) -> C:  # 지연 초기화

    @abstractmethod
    def _get_client(self) -> C:

    @abstractmethod
    def test_connection(self, metadata) -> TestConnectionResult:
```

**연결 빌더** (`connections/builders.py`):

- `create_generic_db_connection()` — SQLAlchemy Engine 생성
- `get_connection_url_common()` — 연결 URL 구성
- `get_connection_args_common()` — 연결 인수 추출

### 소스 계층 구조

```text
Source (IterStep)
  ├── DatabaseServiceSource
  │   └── CommonDbSourceService (SQLAlchemy Inspector 기반)
  │       └── MysqlSource, PostgresSource, SnowflakeSource 등
  ├── DashboardServiceSource
  ├── PipelineServiceSource
  ├── MessagingServiceSource
  └── [기타 서비스 소스]
```

### 싱크 구현

`MetadataRestSink` (`sink/metadata_rest.py`):

- 소스로부터 생성/패치 요청을 수신
- `singledispatchmethod`를 사용하여 엔티티 유형별로 라우팅
- 대량 API 호출을 위해 엔티티를 버퍼링 (배치 크기 설정 가능)
- 속도 제한, 중복 제거, 라이프사이클 업데이트를 처리

### 새 커넥터 추가 방법

1. 디렉토리 생성: `source/database/{connector_name}/`
2. `connection.py` 구현 (`BaseConnection` 상속)
3. `metadata.py` 구현 (`CommonDbSourceService` 상속)
4. `service_spec.py`에 등록 (`ServiceSpec` 선언)
5. `setup.py`에 의존성 추가 (선택적 플러그인 extras)
6. 프레임워크가 `utils/importer.py`를 통해 자동 발견 (동적 로딩)

### 핵심 설계 결정

1. **상속 계층**: 특정 소스가 공통 기본 클래스를 상속
2. **믹스인**: 프로파일링, 리니지, 사용량을 선택적 계층으로 추가
3. **서비스 명세**: 선언적 등록으로 순환 임포트를 방지
4. **동적 로딩**: 플러그인이 필요할 때만 로딩 (지연 임포트)
5. **토폴로지 기반 실행**: 모든 소스 유형에 대한 범용 러너
6. **Either 모나드**: 함수형 에러 처리로 파이프라인 중단을 방지
