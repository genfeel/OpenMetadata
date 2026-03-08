# OpenMetadata 커스터마이징 분석서

> **버전**: 1.12.0-SNAPSHOT
> **분석일**: 2026-03-08
> **목적**: 업무 도입을 위한 커스터마이징 요구사항 분석 및 실행 가이드

---

## 목차

1. [요구사항 매핑 요약](#1-요구사항-매핑-요약)
2. [데이터 흐름 추적: 테이블 조회](#2-데이터-흐름-추적-테이블-조회)
3. [조직/계정/권한 체계 분석](#3-조직계정권한-체계-분석)
4. [Business Glossary 및 분류체계 분석](#4-business-glossary-및-분류체계-분석)
5. [요구사항별 커스터마이징 가이드](#5-요구사항별-커스터마이징-가이드)
6. [Vue 전환 전략](#6-vue-전환-전략)
7. [추천 실행 순서](#7-추천-실행-순서)

---

## 1. 요구사항 매핑 요약

| # | 요구사항 | 난이도 | 수정 대상 모듈 | 성격 |
| -- | ------- | ------ | ------------ | ---- |
| 1 | 조직/계정 마스터 커스터마이징 | 중 | spec(Team/User 스키마), service, ui | 기존 확장 |
| 2 | 조직/마스터 데이터 동기화 | 중 | ingestion(신규 커넥터) 또는 별도 서비스 | 신규 개발 |
| 3 | 권한 체계 커스터마이징 | 상 | spec(Policy 스키마), service(Authorizer), ui | 기존 수정 |
| 4 | UI Vue 변환 | 최상 | ui 전체 재작성 | 신규 개발 |
| 5 | UI 커스터마이징 | 중~상 | ui(컴포넌트, 페이지) | 기존 수정 |
| 6 | DA# 표준단어/용어 동기화, 분류체계 | 상 | spec(신규 스키마), service, ingestion | 신규+확장 |
| 7 | Business Glossary 커스터마이징 | 중 | spec(Glossary 스키마), service, ui | 기존 확장 |
| 8 | 통제용어/택소노미 관리 | 상 | spec(신규 스키마), service, ui | 신규 개발 |
| 9 | IIoT 영상데이터 표준화 | 상 | spec(신규 엔티티), ingestion(신규 커넥터) | 신규 개발 |
| 10 | IoT/로봇 데이터 표준 | 상 | spec(신규 엔티티), ingestion(신규 커넥터) | 신규 개발 |
| 11 | IT+OT 데이터 통합 | 상 | spec, service, ingestion, ui 전반 | 신규 개발 |
| 12 | 데이터 개더링 체계 | 상 | ingestion 확장 또는 별도 시스템 | 신규 개발 |

---

## 2. 데이터 흐름 추적: 테이블 조회

### 전체 흐름도

```text
프론트엔드
┌─────────────────────────────────────┐
│ TableDetailsPageV1.tsx              │
│ → getTableDetailsByFQN(fqn)        │
│ → GET /api/v1/tables/name/{fqn}    │
└──────────────┬──────────────────────┘
               │
백엔드 REST
┌──────────────▼──────────────────────┐
│ TableResource.getByName()           │
│ → @GET @Path("/name/{fqn}")         │
│ → authorizer.authorize() 인가 체크  │
│ → repository.getByName() 호출       │
└──────────────┬──────────────────────┘
               │
리포지토리 계층
┌──────────────▼──────────────────────┐
│ EntityRepository.getByName()        │
│ → findByName() (Guava 캐시 확인)    │
│ → setFieldsInternal() 필드 로딩     │
│ → setInheritedFields() 상속 필드    │
└──────────────┬──────────────────────┘
               │
캐시 계층
┌──────────────▼──────────────────────┐
│ EntityLoaderWithName.load()         │
│ → Redis 외부 캐시 확인              │
│ → 캐시 미스 시 DB 조회              │
└──────────────┬──────────────────────┘
               │
데이터베이스
┌──────────────▼──────────────────────┐
│ EntityDAO.findEntityByName()        │
│ → SELECT json FROM table_entity     │
│   WHERE fqnHash = :name             │
│ → JSON 형식으로 메타데이터 반환      │
└─────────────────────────────────────┘
```

### 계층별 핵심 파일

| 계층 | 파일 | 핵심 메서드/함수 |
| ---- | ---- | -------------- |
| UI 페이지 | `pages/TableDetailsPageV1/TableDetailsPageV1.tsx` | `getTableDetailsByFQN()` |
| UI API | `rest/tableAPI.ts` | `getTableDetailsByFQN(fqn, params)` → `GET /tables/name/{fqn}` |
| REST 엔드포인트 | `resources/databases/TableResource.java` | `getByName()` → `getByNameInternal()` |
| 기본 리소스 | `resources/EntityResource.java` | `getByNameInternal()` — 인가 + 리포지토리 호출 |
| 리포지토리 | `jdbi3/TableRepository.java` | `EntityRepository<Table>` 상속 |
| 기본 리포지토리 | `jdbi3/EntityRepository.java` | `getByName()`, `findByName()`, Guava 캐시 |
| DAO | `jdbi3/EntityDAO.java` | `findEntityByName()` — JDBI SQL 매핑 |
| 엔티티 등록 | `Entity.java` | `registerEntity("table", TableRepository)` |
| DB 스키마 | `bootstrap/sql/schema/mysql.sql` | `table_entity` 테이블 (JSON 컬럼) |

### 핵심 아키텍처 포인트

**모든 엔티티가 동일한 패턴을 따른다:**

1. 메타데이터는 **JSON 컬럼**에 저장 (MySQL의 JSON 타입)
2. `id`, `fqnHash`, `name` 등은 **가상 컬럼**(Generated Column)으로 인덱싱
3. **Guava LoadingCache + Redis**로 2단계 캐시
4. 새로운 엔티티를 추가하려면 이 패턴을 그대로 복제하면 된다

---

## 3. 조직/계정/권한 체계 분석

### 3.1 Team 계층 구조

```text
Organization (최상위)
  └→ BusinessUnit (사업부)
     └→ Division (본부)
        └→ Department (부서)
           └→ Group (그룹)
```

**Team 스키마 핵심 필드** (`entity/teams/team.json`):

| 필드 | 타입 | 설명 |
| ---- | ---- | ---- |
| `teamType` | enum | Organization, BusinessUnit, Division, Department, Group |
| `parents` | EntityReferenceList | 상위 팀 (다중 부모 가능) |
| `children` | EntityReferenceList | 하위 팀 |
| `users` | EntityReferenceList | 팀 멤버 |
| `defaultRoles` | EntityReferenceList | 멤버가 상속하는 기본 역할 |
| `policies` | EntityReferenceList | 팀에 직접 연결된 정책 |

**User 스키마 핵심 필드** (`entity/teams/user.json`):

| 필드 | 타입 | 설명 |
| ---- | ---- | ---- |
| `isAdmin` | boolean | 시스템 관리자 |
| `isBot` | boolean | 자동화용 특수 사용자 |
| `teams` | EntityReferenceList | 소속 팀 |
| `roles` | EntityReferenceList | 직접 할당된 역할 |
| `inheritedRoles` | EntityReferenceList | 팀으로부터 상속된 역할 |
| `personas` | EntityReferenceList | 사용자 페르소나 |
| `domains` | EntityReferenceList | 도메인 접근 권한 |

### 3.2 권한 평가 흐름

```text
요청 발생
    ↓
[DefaultAuthorizer.authorize]
    ↓
SubjectContext 생성 (사용자 정보 로드)
    ├─ User 엔티티 로드 (SubjectCache, 15분 TTL)
    └─ 직접 역할, 팀, isAdmin, domains 포함
    ↓
[특수 사용자 확인]
    ├─ isAdmin() → 모든 권한 즉시 허용
    ├─ isReviewer() → 리소스에 대한 모든 권한 허용
    └─ 일반 사용자 → PolicyEvaluator 실행
    ↓
[PolicyEvaluator.hasPermission]
    ├─ 정책 수집:
    │   ├─ 사용자 직접 역할의 정책
    │   ├─ 소속 팀의 defaultRoles 정책
    │   ├─ 팀 계층 상위의 정책 (재귀적)
    │   └─ 리소스 소유 팀의 정책
    │
    ├─ [1단계: DENY 규칙 평가]
    │   └─ 일치 시 즉시 거부 (DENY 우선)
    │
    └─ [2단계: ALLOW 규칙 평가]
        ├─ 작업별 일치 확인
        ├─ SpEL 조건식 평가 (동적 조건)
        └─ 미허용 작업 → AuthorizationException
    ↓
권한 부여 또는 거부
```

### 3.3 Policy/Rule 구조

**Policy** (`entity/policies/policy.json`):

- `rules`: Rule 배열 (ALLOW/DENY 규칙)
- `roles`: 이 정책을 사용하는 역할
- `teams`: 이 정책을 직접 사용하는 팀

**Rule** (`entity/policies/accessControl/rule.json`):

| 필드 | 타입 | 설명 |
| ---- | ---- | ---- |
| `effect` | enum | `allow` 또는 `deny` |
| `operations` | MetadataOperation[] | 작업 목록 (70종 이상, `*` 와일드카드) |
| `resources` | string[] | 엔티티 타입 (`*` 와일드카드) |
| `condition` | string | SpEL 표현식 (선택적 동적 조건) |

### 3.4 권한 체계 핵심 파일

| 구분 | 파일 | 역할 |
| ---- | ---- | ---- |
| 스키마 | `entity/teams/team.json` | Team 엔티티 정의 |
| 스키마 | `entity/teams/user.json` | User 엔티티 정의 |
| 스키마 | `entity/teams/role.json` | Role 엔티티 정의 |
| 스키마 | `entity/policies/policy.json` | Policy 엔티티 정의 |
| 스키마 | `entity/policies/accessControl/rule.json` | Rule 정의 |
| 인가 | `security/Authorizer.java` | 인가 인터페이스 |
| 인가 | `security/DefaultAuthorizer.java` | 기본 인가 구현체 |
| 정책평가 | `security/policyevaluator/PolicyEvaluator.java` | DENY→ALLOW 규칙 평가 |
| 컨텍스트 | `security/policyevaluator/SubjectContext.java` | 사용자 권한 맥락 |
| 캐시 | `security/policyevaluator/SubjectCache.java` | 정책/사용자 캐시 (2분/15분 TTL) |
| 리포지토리 | `jdbi3/UserRepository.java` | User DB 접근 |
| 리포지토리 | `jdbi3/TeamRepository.java` | Team DB 접근 |

### 3.5 도메인 기반 접근 제어

- 사용자와 리소스 모두 도메인(domains) 목록을 보유
- **계층 구조 지원**: 부모 도메인 사용자 → 자식 도메인 리소스 접근 가능
- 예: "Engineering" 도메인 사용자 → "Engineering.Backend.Services" 리소스 접근 가능
- Admin/Bot은 도메인 제한 없음

---

## 4. Business Glossary 및 분류체계 분석

### 4.1 Glossary vs Classification 비교

| 측면 | Glossary (용어집) | Classification (분류체계) |
| ---- | ---------------- | ---------------------- |
| **목적** | 비즈니스 용어 정의 | 데이터 카테고리/레이블 |
| **구조** | Glossary → GlossaryTerm (계층) | Classification → Tag (계층) |
| **사용처** | 메타데이터 설명, 비즈니스 정의 | 엔티티/필드 분류, 거버넌스 |
| **TagLabel.source** | `"Glossary"` | `"Classification"` |
| **상호배타성** | 용어별 설정 | 분류별 설정 |
| **자동화** | 없음 | 자동 분류 (recognizers) |

### 4.2 Glossary 스키마 구조

**Glossary** (`entity/data/glossary.json`) 핵심 필드:

| 필드 | 설명 |
| ---- | ---- |
| `name` | 용어집 이름 (유니크) |
| `termCount` | 모든 하위 용어 총 수 |
| `reviewers` | 검토자 목록 |
| `mutuallyExclusive` | 직계 자식 용어가 상호배타적인지 |
| `tags` | 적용된 태그 라벨 |

**GlossaryTerm** (`entity/data/glossaryTerm.json`) 핵심 필드:

| 필드 | 설명 |
| ---- | ---- |
| `glossary` | 소속 용어집 (필수) |
| `parent` | 상위 용어 (null이면 루트) |
| `children` | 하위 용어 |
| `synonyms` | 동의어 배열 |
| `relatedTerms` | 관련 용어 |
| `references` | 외부 용어집 링크 |
| `fullyQualifiedName` | `glossaryName.parentTerm.childTerm` 형식 |

### 4.3 Classification/Tag 스키마 구조

**Classification** (`entity/classification/classification.json`) 핵심 필드:

| 필드 | 설명 |
| ---- | ---- |
| `termCount` | 자식 Tag 총 수 |
| `mutuallyExclusive` | 태그 상호배타성 |
| `autoClassificationConfig` | 자동 분류 설정 (enabled, confidence, conflictResolution) |

**Tag** (`entity/classification/tag.json`) 핵심 필드:

| 필드 | 설명 |
| ---- | ---- |
| `classification` | 소속 분류 |
| `parent` | 상위 태그 (계층 구조) |
| `recognizers` | 자동 인식 설정 |
| `autoClassificationEnabled` | 자동 분류 활성화 |
| `autoClassificationPriority` | 우선순위 (0~100) |

**TagLabel** (`type/tagLabel.json`) — 엔티티에 적용되는 태그 표현:

| 필드 | 설명 |
| ---- | ---- |
| `tagFQN` | 태그 정규화된 이름 |
| `source` | `"Classification"` 또는 `"Glossary"` |
| `labelType` | Manual, Propagated, Automated, Derived, Generated |
| `state` | Suggested, Confirmed |

### 4.4 Glossary/Classification 핵심 파일

| 구분 | 파일 | 역할 |
| ---- | ---- | ---- |
| 스키마 | `entity/data/glossary.json` | Glossary 엔티티 |
| 스키마 | `entity/data/glossaryTerm.json` | GlossaryTerm 엔티티 |
| 스키마 | `entity/classification/classification.json` | Classification 엔티티 |
| 스키마 | `entity/classification/tag.json` | Tag 엔티티 |
| 스키마 | `type/tagLabel.json` | 엔티티에 적용되는 태그 표현 |
| 리소스 | `resources/glossary/GlossaryResource.java` | `/v1/glossaries` API |
| 리소스 | `resources/glossary/GlossaryTermResource.java` | `/v1/glossaryTerms` API |
| 리소스 | `resources/tags/ClassificationResource.java` | `/v1/classifications` API |
| 리소스 | `resources/tags/TagResource.java` | `/v1/tags` API |
| 리포지토리 | `jdbi3/GlossaryRepository.java` | Glossary DB (CSV 지원) |
| 리포지토리 | `jdbi3/GlossaryTermRepository.java` | GlossaryTerm DB (계층, 이동) |
| UI 페이지 | `pages/Glossary/GlossaryPage/` | Glossary 메인 페이지 |
| UI 컴포넌트 | `components/Glossary/GlossaryV1.component.tsx` | Glossary 메인 컴포넌트 |
| UI 상태 | `components/Glossary/useGlossary.store.ts` | Zustand 스토어 |
| UI API | `rest/glossaryAPI.ts` | Glossary REST 클라이언트 |

---

## 5. 요구사항별 커스터마이징 가이드

### #1. 조직/계정 마스터 커스터마이징

**현재 구조**: Team(5단계 계층) + User + Role + Persona

**커스터마이징 방향**:

- Team 스키마에 커스텀 필드 추가 (사업자번호, 부서코드 등) → `team.json` 수정
- User 스키마에 사번, 직급 등 추가 → `user.json` 수정
- `TeamRepository`, `UserRepository`에서 추가 필드 처리
- 코드 재생성 필요: `mvn clean install` (Java), `make generate` (Python)

**수정 파일**:

- `openmetadata-spec/.../entity/teams/team.json`
- `openmetadata-spec/.../entity/teams/user.json`
- `openmetadata-service/.../jdbi3/TeamRepository.java`
- `openmetadata-service/.../jdbi3/UserRepository.java`

### #2. 조직/마스터 데이터 동기화

**방안 A — 인제스천 커넥터 방식**:

- 신규 Python 커넥터 개발 (HR 시스템, AD 등에서 조직/사용자 동기화)
- `ingestion/src/metadata/ingestion/source/metadata/` 하위에 구현
- OpenMetadata REST API (`/v1/teams`, `/v1/users`)로 데이터 전송

**방안 B — 별도 동기화 서비스**:

- 스케줄러 기반 독립 서비스 (Spring Batch, Airflow DAG 등)
- OpenMetadata SDK를 통해 API 호출
- 양방향 동기화가 필요하면 이 방식이 적합

### #3. 권한 체계 커스터마이징

**현재 구조**: Role → Policy → Rule (DENY 우선, SpEL 조건식)

**커스터마이징 방향**:

- 신규 MetadataOperation 추가 → `accessControl/rule.json`의 operations enum 확장
- 커스텀 SpEL 함수 추가 → `PolicyEvaluator.java` 수정
- 도메인 기반 접근 제어 확장 → `SubjectContext.hasDomains()` 수정
- 새로운 Authorizer 구현체 작성 (기존 DefaultAuthorizer를 래핑하거나 교체)

**수정 파일**:

- `openmetadata-spec/.../accessControl/rule.json`
- `openmetadata-service/.../security/policyevaluator/PolicyEvaluator.java`
- `openmetadata-service/.../security/policyevaluator/SubjectContext.java`
- `openmetadata-service/.../security/DefaultAuthorizer.java`

### #6. DA# 표준단어/용어 동기화, 분류체계

**접근 방식**:

- Classification 시스템을 활용하여 "주제영역" 분류체계 구현
- 신규 인제스천 커넥터로 DA# 레파지토리에서 표준단어/표준용어를 동기화
- GlossaryTerm의 `synonyms` 필드를 활용하여 표준어-유사어 매핑
- Tag 자동 분류 기능 활용 (`autoClassificationConfig`)

### #7. Business Glossary 커스터마이징

**일반 사용자용 vs 정보팀용 분리**:

- Glossary 엔티티를 역할별로 분리 (일반용, 정보팀용 각각 생성)
- 권한 체계(Policy/Rule)로 접근 제어
- GlossaryTerm 스키마에 커스텀 필드 추가 (승인 상태, 담당자 등)
- `reviewers` 필드를 활용한 검토 워크플로우

### #8. 통제용어/택소노미 관리

**동음이의어/이음동의어 관리**:

- GlossaryTerm의 `synonyms` 필드 활용 (이음동의어)
- GlossaryTerm의 `relatedTerms` 필드 활용 (관련 용어)
- 동음이의어는 별도 커스텀 필드 또는 신규 스키마 필요
- 택소노미: Classification의 계층적 Tag 구조 활용

**필요시 신규 스키마 개발**:

- `entity/data/controlledVocabulary.json` 등 신규 엔티티 정의
- 대응하는 Resource, Repository, DAO 추가

### #9~#10. IIoT 영상/IoT/로봇 데이터 표준화

**접근 방식**:

- 신규 엔티티 스키마 정의 (예: `videoAsset.json`, `iotDataStream.json`)
- 신규 서비스 타입 추가 (예: `IoTService`, `VideoService`)
- 신규 인제스천 커넥터 개발 (IoT 플랫폼, 영상 시스템 연동)
- 표준 메타데이터 속성 정의 (해상도, 프레임레이트, 센서타입 등)

**수정/추가 파일**:

- `openmetadata-spec/.../entity/data/` — 신규 엔티티 스키마
- `openmetadata-spec/.../entity/services/` — 신규 서비스 스키마
- `openmetadata-service/.../resources/` — 신규 REST 리소스
- `openmetadata-service/.../jdbi3/` — 신규 리포지토리
- `ingestion/.../source/` — 신규 커넥터

### #11. IT+OT 데이터 통합

**접근 방식**:

- IT 데이터(기존 DB/대시보드 등)와 OT 데이터(IoT/영상)를 통합하는 리니지 확장
- Domain 엔티티를 활용하여 IT/OT 영역 분리 및 통합 관리
- 통합 검색을 위한 Elasticsearch 인덱스 확장
- 크로스 도메인 리니지 시각화

### #12. 데이터 개더링 체계

**접근 방식**:

- 인제스천 프레임워크의 토폴로지 기반 실행 패턴 활용
- 다양한 소스(IT/OT/IoT)에서 메타데이터를 수집하는 통합 워크플로우
- Airflow DAG 기반 스케줄링
- 수집 현황 모니터링 대시보드 (UI 커스터마이징)

---

## 6. Vue 전환 전략

### 선택지 비교

| 방안 | 장점 | 단점 |
| ---- | ---- | ---- |
| **A. React 유지** | 업스트림 업데이트 반영, 빠른 시작 | Vue 전환 불가 |
| **B. Vue 완전 재작성** | 자유로운 UI 설계 | 막대한 공수, 업스트림 단절 |
| **C. API만 활용 + Vue 신규 (추천)** | 백엔드 재활용, UI 자유도 확보 | 프론트엔드 전체 신규 |

### C안 (추천) 상세

**백엔드 REST API는 그대로 활용**하고 Vue 프론트엔드를 새로 만든다:

- OpenMetadata의 REST API는 55개 이상의 잘 정의된 엔드포인트 보유
- Swagger/OpenAPI 문서로 Vue API 클라이언트 자동 생성 가능
- 기존 77개 React API 파일(`src/rest/`)의 구조를 참고하여 Vue 대응 코드 작성
- 백엔드 커스터마이징(#1,2,3,6~12)은 업스트림 변경과 병행 유지 가능

**Vue 프로젝트 구조 제안**:

```text
openmetadata-vue/
├── src/
│   ├── api/           # REST 클라이언트 (기존 rest/ 참고)
│   ├── components/    # Vue 컴포넌트
│   ├── views/         # 페이지 (기존 pages/ 참고)
│   ├── stores/        # Pinia 상태 관리 (기존 Zustand 참고)
│   ├── composables/   # Vue Composables (기존 hooks/ 참고)
│   ├── types/         # 타입 (기존 generated/ 재활용)
│   ├── i18n/          # 다국어 (기존 locale/ 재활용)
│   └── router/        # Vue Router
└── package.json
```

**재활용 가능한 자산**:

- `generated/` TypeScript 타입 정의 → 그대로 사용
- `locale/` i18n 번역 파일 → 그대로 사용
- `rest/` API 호출 패턴 → Vue 스타일로 변환
- Swagger 문서 → API 클라이언트 자동 생성

---

## 7. 추천 실행 순서

```text
[1단계] 기반 구축 (1~2개월)
  ├─ 로컬 환경 구동 (Docker)
  ├─ REST API 전수 조사 (Swagger UI)
  ├─ 데이터 흐름 추적 학습
  ├─ #1 조직/계정 스키마 확장 (가장 간단한 커스터마이징으로 패턴 학습)
  └─ #2 마스터 데이터 동기화 프로토타입

[2단계] 권한 및 거버넌스 (2~3개월)
  ├─ #3 권한 체계 커스터마이징
  ├─ #7 Business Glossary 확장 (일반용/정보팀용)
  └─ #8 통제용어/택소노미 관리

[3단계] 데이터 표준 체계 (2~3개월)
  ├─ #6 DA# 표준단어/용어 동기화
  ├─ #9 IIoT 영상데이터 표준화
  └─ #10 IoT/로봇 데이터 표준

[4단계] 통합 및 확장 (2~3개월)
  ├─ #11 IT+OT 데이터 통합
  └─ #12 데이터 개더링 체계

[5단계] Vue 프론트엔드 (병렬 진행, 3~6개월)
  ├─ #4 Vue 프론트엔드 기반 구축
  └─ #5 UI 커스터마이징 (각 단계 결과물 반영)
```

**핵심 원칙**:

- 1단계에서 패턴을 완전히 이해한 후 진행
- 스키마 변경(spec) → 코드 재생성 → 백엔드 수정 → UI 반영 순서 준수
- Vue 전환은 다른 작업과 병렬로 진행 가능 (API가 독립적이므로)
- 각 단계에서 업스트림 변경사항 머지 전략 수립 필요
