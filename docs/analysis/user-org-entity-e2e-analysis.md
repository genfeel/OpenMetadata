# OpenMetadata 사용자 및 조직 엔티티 End-to-End 분석

> 분석 대상: OpenMetadata (main branch)
> 분석 일자: 2026-03-09
> 범위: 사용자(User), 팀/조직(Team), 역할(Role), 정책(Policy), 봇(Bot) 엔티티의 스키마 정의 → DB 저장 → API 입출력 전 과정

---

## 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [데이터베이스 테이블 구조](#2-데이터베이스-테이블-구조)
3. [JSON Schema 정의 (엔티티 모델)](#3-json-schema-정의-엔티티-모델)
4. [코드 레이어 구조](#4-코드-레이어-구조)
5. [API 엔드포인트 목록](#5-api-엔드포인트-목록)
6. [시드 데이터 (초기 데이터)](#6-시드-데이터-초기-데이터)
7. [실제 API 호출 예시](#7-실제-api-호출-예시)
8. [데이터 흐름 (End-to-End)](#8-데이터-흐름-end-to-end)
9. [소스 코드 위치 인덱스](#9-소스-코드-위치-인덱스)

---

## 1. 아키텍처 개요

OpenMetadata는 **Schema-First** 접근 방식을 사용한다. JSON Schema로 엔티티를 정의하고, 이를 기반으로 Java 모델과 TypeScript 타입이 생성된다. 데이터베이스에는 **EAV(Entity-Attribute-Value) 패턴**으로 저장되며, 각 테이블의 `json` (JSONB) 컬럼에 전체 엔티티 데이터가 들어가고, 검색/인덱싱에 필요한 필드만 `GENERATED ALWAYS AS` 가상 컬럼으로 추출된다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        클라이언트 (UI / CLI / SDK)                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ HTTP (REST API)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Resource Layer (JAX-RS)                                            │
│  - UserResource.java (/v1/users)                                    │
│  - TeamResource.java (/v1/teams)                                    │
│  - RoleResource.java (/v1/roles)                                    │
│  - BotResource.java  (/v1/bots)                                     │
│  - PolicyResource.java (/v1/policies)                               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Mapper Layer (DTO → Entity 변환)                                    │
│  - UserMapper.java   (CreateUser → User)                            │
│  - TeamMapper.java   (CreateTeam → Team)                            │
│  - RoleMapper.java   (CreateRole → Role)                            │
│  - BotMapper.java    (CreateBot  → Bot)                             │
│  - PolicyMapper.java (CreatePolicy → Policy)                        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Repository Layer (비즈니스 로직 + 관계 관리)                          │
│  - UserRepository.java                                              │
│  - TeamRepository.java                                              │
│  - RoleRepository.java                                              │
│  - BotRepository.java                                               │
│  - PolicyRepository.java                                            │
│  - TokenRepository.java                                             │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  DAO Layer (CollectionDAO.java 내부 인터페이스)                       │
│  - UserDAO    → user_entity 테이블                                   │
│  - TeamDAO    → team_entity 테이블                                   │
│  - RoleDAO    → role_entity 테이블                                   │
│  - BotDAO     → bot_entity 테이블                                    │
│  - PolicyDAO  → policy_entity 테이블                                 │
│  - TokenDAO   → user_tokens 테이블                                   │
│  - (공통)      → entity_relationship 테이블                           │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ JDBI (SQL)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PostgreSQL                                                         │
│  user_entity │ team_entity │ role_entity │ policy_entity │           │
│  bot_entity  │ user_tokens │ entity_relationship                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 데이터베이스 테이블 구조

> 소스: `bootstrap/sql/schema/postgres.sql`

### 2.1 user_entity (사용자)

| 컬럼 | 타입 | 생성 방식 | 설명 |
|------|------|-----------|------|
| `id` | varchar(36) | `GENERATED ALWAYS AS (json->>'id')` | UUID PK |
| `name` | varchar(256) | `GENERATED ALWAYS AS (json->>'name')` | 사용자 고유 이름 |
| `email` | varchar(256) | `GENERATED ALWAYS AS (json->>'email')` | 이메일 |
| `deactivated` | varchar(8) | `GENERATED ALWAYS AS (json->>'deactivated')` | 비활성화 여부 |
| `json` | jsonb | 직접 저장 | **전체 사용자 데이터** |
| `updatedat` | bigint | `GENERATED ALWAYS AS (json->>'updatedAt')` | 수정 시각 (epoch ms) |
| `updatedby` | varchar(256) | `GENERATED ALWAYS AS (json->>'updatedBy')` | 수정자 |
| `deleted` | boolean | `GENERATED ALWAYS AS (json->>'deleted')` | 소프트 삭제 |
| `namehash` | varchar(256) | 직접 저장 | 이름 해시 (유니크 인덱스) |

### 2.2 team_entity (팀/조직)

| 컬럼 | 타입 | 생성 방식 | 설명 |
|------|------|-----------|------|
| `id` | varchar(36) | `GENERATED ALWAYS AS (json->>'id')` | UUID PK |
| `name` | varchar(256) | `GENERATED ALWAYS AS (json->>'name')` | 팀 고유 이름 |
| `json` | jsonb | 직접 저장 | **전체 팀 데이터** |
| `updatedat` | bigint | `GENERATED ALWAYS AS (json->>'updatedAt')` | 수정 시각 |
| `updatedby` | varchar(256) | `GENERATED ALWAYS AS (json->>'updatedBy')` | 수정자 |
| `deleted` | boolean | `GENERATED ALWAYS AS (json->>'deleted')` | 소프트 삭제 |
| `teamtype` | varchar(64) | `GENERATED ALWAYS AS (json->>'teamType')` | 팀 유형 |
| `namehash` | varchar(256) | 직접 저장 | 이름 해시 |

### 2.3 role_entity (역할)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | varchar(36) | UUID PK |
| `name` | varchar(256) | 역할 이름 |
| `json` | jsonb | 전체 역할 데이터 |
| `updatedat` | bigint | 수정 시각 |
| `updatedby` | varchar(256) | 수정자 |
| `deleted` | boolean | 소프트 삭제 |
| `namehash` | varchar(256) | 이름 해시 |

### 2.4 policy_entity (정책)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | varchar(36) | UUID PK |
| `name` | varchar(256) | 정책 이름 |
| `json` | jsonb | 전체 정책 데이터 (rules 포함) |
| `updatedat` | bigint | 수정 시각 |
| `updatedby` | varchar(256) | 수정자 |
| `deleted` | boolean | 소프트 삭제 |
| `fqnhash` | varchar(768) | FQN 해시 |

### 2.5 bot_entity (봇)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | varchar(36) | UUID PK |
| `name` | varchar(256) | 봇 이름 |
| `json` | jsonb | 전체 봇 데이터 |
| `updatedat` | bigint | 수정 시각 |
| `updatedby` | varchar(256) | 수정자 |
| `deleted` | boolean | 소프트 삭제 |
| `namehash` | varchar(256) | 이름 해시 |

### 2.6 user_tokens (인증 토큰)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `token` | varchar(36) | 토큰 값 (PK) |
| `userid` | varchar(36) | user_entity.id 참조 |
| `tokentype` | varchar(50) | `EMAIL_VERIFICATION`, `PASSWORD_RESET`, `REFRESH_TOKEN`, `PERSONAL_ACCESS_TOKEN` |
| `json` | jsonb | 전체 토큰 데이터 |
| `expirydate` | bigint | 만료일 (epoch ms) |

### 2.7 entity_relationship (엔티티 간 관계 — 범용)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `fromid` | varchar(36) | 출발 엔티티 ID |
| `toid` | varchar(36) | 도착 엔티티 ID |
| `fromentity` | varchar(256) | 출발 엔티티 타입 (`user`, `team`, `role` 등) |
| `toentity` | varchar(256) | 도착 엔티티 타입 |
| `relation` | smallint | 관계 유형 (ordinal 값) |
| `jsonschema` | varchar(256) | JSON 스키마 (선택) |
| `json` | jsonb | 관계 메타데이터 (선택) |
| `deleted` | boolean | 소프트 삭제 |

**관계 유형 (relation ordinal):**

| ordinal | Enum | 설명 | 사용 예 |
|---------|------|------|---------|
| 0 | CONTAINS | 포함 | bot → user |
| 8 | OWNS | 소유 | user/team → asset |
| 9 | PARENT_OF | 부모-자식 | team → team (계층) |
| 10 | HAS | 보유 | team → user, user → role, team → policy |
| 14 | APPLIED_TO | 적용 | policy → entity |
| 20 | DEFAULTS_TO | 기본값 | team → defaultRole |

### 2.8 테이블 간 관계 다이어그램

```
                          ┌──────────────┐
                          │ policy_entity│
                          └──────┬───────┘
                                 │ HAS (10)
                                 │
┌─────────────┐   HAS (10)    ┌──┴───────────┐   PARENT_OF (9)   ┌──────────────┐
│ role_entity │◄──────────────│ team_entity  │──────────────────►│ team_entity  │
└──────┬──────┘               └──────┬───────┘   (계층: Org→BU    │ (자식 팀)     │
       │                             │            →Div→Dept→Grp)  └──────────────┘
       │ HAS (10)                    │ HAS (10)
       │                             │
       │         ┌───────────────────┘
       │         │
       ▼         ▼
  ┌──────────────┐        CONTAINS (0)     ┌────────────┐
  │ user_entity  │◄────────────────────────│ bot_entity │
  └──────┬───────┘                         └────────────┘
         │
         │ (userid FK)
         ▼
  ┌──────────────┐
  │ user_tokens  │
  └──────────────┘
```

---

## 3. JSON Schema 정의 (엔티티 모델)

> 소스: `openmetadata-spec/src/main/resources/json/schema/`

### 3.1 User 엔티티

**스키마 위치:** `entity/teams/user.json`
**Java 클래스:** `org.openmetadata.schema.entity.teams.User`
**필수 필드:** `id`, `name`, `email`

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | 고유 식별자 |
| `name` | entityName | 고유 이름 (LDAP uid 등) |
| `fullyQualifiedName` | string | FQN (name과 동일) |
| `email` | email | 이메일 주소 |
| `displayName` | string | 표시 이름 (예: "홍길동") |
| `description` | markdown | 자기소개 |
| `isBot` | boolean | 봇 여부 (기본: false) |
| `isAdmin` | boolean | 관리자 여부 (기본: false) |
| `teams` | entityReferenceList | 소속 팀 목록 |
| `roles` | entityReferenceList | 할당된 역할 목록 |
| `inheritedRoles` | entityReferenceList | 팀에서 상속받은 역할 |
| `personas` | entityReferenceList | 할당된 페르소나 |
| `defaultPersona` | entityReference | 기본 페르소나 |
| `profile` | Profile | 프로필 이미지 등 |
| `timezone` | string | 시간대 |
| `authenticationMechanism` | object | 인증 방식 (`JWT` / `SSO` / `BASIC`) |
| `domains` | entityReferenceList | 소속 도메인 |
| `deleted` | boolean | 소프트 삭제 여부 |
| `lastLoginTime` | timestamp | 마지막 로그인 시각 |
| `lastActivityTime` | timestamp | 마지막 활동 시각 |

### 3.2 Team 엔티티

**스키마 위치:** `entity/teams/team.json`
**Java 클래스:** `org.openmetadata.schema.entity.teams.Team`
**필수 필드:** `id`, `name`

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | uuid | 고유 식별자 |
| `name` | entityName | 팀 고유 이름 |
| `teamType` | enum | `Organization`, `BusinessUnit`, `Division`, `Department`, `Group` |
| `displayName` | string | 표시 이름 |
| `email` | email | 팀 이메일 |
| `description` | markdown | 팀 설명 |
| `parents` | entityReferenceList | 상위 팀 |
| `children` | entityReferenceList | 하위 팀 |
| `users` | entityReferenceList | 소속 사용자 |
| `userCount` | integer | 소속 사용자 수 |
| `childrenCount` | integer | 하위 팀 수 |
| `defaultRoles` | entityReferenceList | 기본 역할 (소속 사용자 자동 상속) |
| `policies` | entityReferenceList | 연결된 정책 |
| `owners` | entityReferenceList | 팀 소유자 |
| `isJoinable` | boolean | 가입 가능 여부 (기본: true) |
| `domains` | entityReferenceList | 소속 도메인 |
| `defaultPersona` | entityReference | 기본 페르소나 (Group만 해당) |
| `deleted` | boolean | 소프트 삭제 여부 |

**팀 계층 구조 및 허용 관계:**

```
Organization (최상위, 시스템 자동 생성, 1개만 존재)
 ├── BusinessUnit (사업부)
 │    ├── BusinessUnit
 │    ├── Division (부문)
 │    ├── Department (부서)
 │    └── Group (그룹)
 ├── Division
 │    ├── Division
 │    ├── Department
 │    └── Group
 ├── Department
 │    ├── Department
 │    └── Group
 └── Group (최하위, 사용자만 포함, 하위 팀 불가)
```

### 3.3 CreateUser 요청 DTO

**스키마 위치:** `api/teams/createUser.json`
**필수 필드:** `name`, `email`

| 필드 | 타입 | 설명 |
|------|------|------|
| `name` | entityName | 사용자 이름 |
| `email` | email | 이메일 |
| `displayName` | string | 표시 이름 |
| `description` | markdown | 자기소개 |
| `isBot` | boolean | 봇 여부 |
| `isAdmin` | boolean | 관리자 여부 |
| `teams` | uuid[] | 소속 팀 ID 목록 |
| `roles` | uuid[] | 역할 ID 목록 |
| `personas` | entityReferenceList | 페르소나 목록 |
| `profile` | Profile | 프로필 |
| `timezone` | string | 시간대 |
| `password` | string | 비밀번호 |
| `confirmPassword` | string | 비밀번호 확인 |
| `domains` | entityName[] | 도메인 이름 목록 |

### 3.4 CreateTeam 요청 DTO

**스키마 위치:** `api/teams/createTeam.json`
**필수 필드:** `name`, `teamType`

| 필드 | 타입 | 설명 |
|------|------|------|
| `name` | entityName | 팀 이름 |
| `teamType` | enum | 팀 유형 (필수) |
| `displayName` | string | 표시 이름 |
| `email` | email | 팀 이메일 |
| `description` | markdown | 팀 설명 |
| `parents` | uuid[] | 상위 팀 ID (미지정 시 Organization이 기본 부모) |
| `children` | uuid[] | 하위 팀 ID |
| `users` | uuid[] | 소속 사용자 ID |
| `defaultRoles` | uuid[] | 기본 역할 ID |
| `policies` | uuid[] | 정책 ID |
| `owners` | entityReferenceList | 소유자 |
| `isJoinable` | boolean | 가입 가능 여부 |
| `domains` | entityName[] | 도메인 이름 |

---

## 4. 코드 레이어 구조

### 4.1 Resource Layer (REST 엔드포인트)

Dropwizard(JAX-RS) 기반. HTTP 요청 수신, 인증/인가 처리, Mapper/Repository 호출.

| 클래스 | 경로 | Base Path |
|--------|------|-----------|
| `UserResource` | `resources/teams/UserResource.java` | `/v1/users` |
| `TeamResource` | `resources/teams/TeamResource.java` | `/v1/teams` |
| `RoleResource` | `resources/teams/RoleResource.java` | `/v1/roles` |
| `BotResource` | `resources/bots/BotResource.java` | `/v1/bots` |
| `PolicyResource` | `resources/policies/PolicyResource.java` | `/v1/policies` |

### 4.2 Mapper Layer (DTO → Entity 변환)

CreateXxx 요청 DTO를 엔티티 객체로 변환하며, 이 과정에서 입력값 검증과 참조 해석을 수행한다.

| 클래스 | 주요 변환 로직 |
|--------|---------------|
| `UserMapper` | name/email 소문자 변환, teams/roles UUID → EntityReference 검증 |
| `TeamMapper` | teamType 검증, Organization/Group 생성 제약, 부모-자식 계층 검증 |
| `RoleMapper` | 최소 1개 정책 보유 검증, 정책 참조 해석 |
| `BotMapper` | botUser 존재 확인, 중복 봇-사용자 연결 차단 |
| `PolicyMapper` | 규칙 할당, enabled 상태 관리 |

### 4.3 Repository Layer (비즈니스 로직)

엔티티 CRUD, 관계(entity_relationship) 관리, 필드 조회 시 관련 데이터 조합.

| 클래스 | 주요 메서드 |
|--------|-------------|
| `UserRepository` | `fetchAndSetTeams()`, `fetchAndSetRoles()`, `checkEmailAlreadyExists()`, `updateLastActivityTime()`, `importCsv()`, `exportCsv()` |
| `TeamRepository` | `initOrganization()`, `listHierarchy()`, `updateTeamUsers()`, `populateParents()`, `populateChildren()`, `bulkAddAssets()` |
| `RoleRepository` | `fetchAndSetPolicies()`, `fetchAndSetTeams()`, `fetchAndSetUsers()` |
| `BotRepository` | `getBotUser()`, `storeRelationships()` (bot→user CONTAINS 관계) |
| `PolicyRepository` | `fetchAndSetTeams()`, `fetchAndSetRoles()` |
| `TokenRepository` | `findByToken()`, `insertToken()`, `deleteToken()`, `findByUserIdAndType()` |

### 4.4 DAO Layer (SQL 실행)

`CollectionDAO.java` 내부에 JDBI 인터페이스로 정의. 각 엔티티별 테이블 접근.

| DAO | 테이블 | 주요 쿼리 |
|-----|--------|-----------|
| `UserDAO` | `user_entity` | `checkEmailExists()`, `findUserByEmail()`, `findUserByNameAndEmail()`, `updateLastActivityTime()`, `getMaxLastActivityTime()` |
| `TeamDAO` | `team_entity` | 표준 CRUD + 계층 쿼리 |
| `RoleDAO` | `role_entity` | 표준 CRUD |
| `BotDAO` | `bot_entity` | 표준 CRUD |
| `PolicyDAO` | `policy_entity` | 표준 CRUD |
| `TokenDAO` | `user_tokens` | `findByToken()`, `getAllUserTokenWithType()`, `insert()`, `delete()`, `deleteTokenByUserAndType()` |

---

## 5. API 엔드포인트 목록

### 5.1 User API (`/v1/users`)

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/v1/users` | 사용자 목록 (팀/관리자/봇 필터, 페이지네이션) |
| `GET` | `/v1/users/{id}` | ID로 사용자 조회 |
| `GET` | `/v1/users/name/{name}` | 이름으로 사용자 조회 |
| `GET` | `/v1/users/loggedInUser` | 현재 로그인 사용자 |
| `GET` | `/v1/users/loggedInUser/groupTeams` | 로그인 사용자의 소속 그룹 팀 |
| `GET` | `/v1/users/{id}/versions` | 사용자 버전 이력 |
| `GET` | `/v1/users/{id}/versions/{version}` | 특정 버전 조회 |
| `GET` | `/v1/users/{id}/assets` | 사용자 소유 자산 |
| `GET` | `/v1/users/name/{name}/assets` | 이름으로 사용자 자산 조회 |
| `GET` | `/v1/users/online` | 온라인 사용자 목록 (관리자) |
| `GET` | `/v1/users/token/{id}` | 봇 사용자 JWT 토큰 |
| `GET` | `/v1/users/auth-mechanism/{id}` | 봇 인증 메커니즘 |
| `GET` | `/v1/users/security/token` | 개인 액세스 토큰 목록 |
| `GET` | `/v1/users/export` | 사용자 CSV 내보내기 |
| `POST` | `/v1/users` | 사용자 생성 |
| `POST` | `/v1/users/signup` | 회원가입 |
| `POST` | `/v1/users/login` | 로그인 |
| `POST` | `/v1/users/logout` | 로그아웃 |
| `POST` | `/v1/users/refresh` | 토큰 갱신 |
| `POST` | `/v1/users/checkEmailVerified` | 이메일 인증 확인 |
| `POST` | `/v1/users/generatePasswordResetLink` | 비밀번호 리셋 링크 |
| `POST` | `/v1/users/password/reset` | 비밀번호 리셋 |
| `PUT` | `/v1/users` | 사용자 생성/수정 (upsert) |
| `PUT` | `/v1/users/changePassword` | 비밀번호 변경 |
| `PUT` | `/v1/users/security/token` | 개인 액세스 토큰 생성 |
| `PUT` | `/v1/users/security/token/revoke` | 개인 액세스 토큰 폐기 |
| `PUT` | `/v1/users/restore` | 삭제된 사용자 복원 |
| `PUT` | `/v1/users/import` | CSV 일괄 가져오기 |
| `PUT` | `/v1/users/registrationConfirmation` | 이메일 인증 확인 |
| `PATCH` | `/v1/users/{id}` | JSON Patch 부분 수정 |
| `DELETE` | `/v1/users/{id}` | 사용자 삭제 |
| `DELETE` | `/v1/users/name/{name}` | 이름으로 삭제 |

### 5.2 Team API (`/v1/teams`)

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/v1/teams` | 팀 목록 (parent, joinable 필터) |
| `GET` | `/v1/teams/hierarchy` | 팀 계층 구조 |
| `GET` | `/v1/teams/{id}` | ID로 팀 조회 |
| `GET` | `/v1/teams/name/{name}` | 이름으로 팀 조회 |
| `GET` | `/v1/teams/{id}/versions` | 팀 버전 이력 |
| `GET` | `/v1/teams/{id}/assets` | 팀 소유 자산 |
| `GET` | `/v1/teams/name/{fqn}/assets` | FQN으로 팀 자산 조회 |
| `GET` | `/v1/teams/assets/counts` | 전체 팀별 자산 수 |
| `GET` | `/v1/teams/name/{name}/export` | 팀 CSV 내보내기 |
| `POST` | `/v1/teams` | 팀 생성 |
| `PUT` | `/v1/teams` | 팀 생성/수정 (upsert) |
| `PUT` | `/v1/teams/{teamId}/users` | 팀 사용자 갱신 |
| `PUT` | `/v1/teams/{name}/assets/add` | 자산 일괄 추가 |
| `PUT` | `/v1/teams/{name}/assets/remove` | 자산 일괄 제거 |
| `PUT` | `/v1/teams/restore` | 삭제된 팀 복원 |
| `PUT` | `/v1/teams/name/{name}/import` | CSV 일괄 가져오기 |
| `PATCH` | `/v1/teams/{id}` | JSON Patch 부분 수정 |
| `PATCH` | `/v1/teams/name/{fqn}` | FQN으로 JSON Patch 수정 |
| `DELETE` | `/v1/teams/{id}` | 팀 삭제 |
| `DELETE` | `/v1/teams/name/{name}` | 이름으로 삭제 |
| `DELETE` | `/v1/teams/{teamId}/users/{userId}` | 팀에서 사용자 제거 |

### 5.3 Role API (`/v1/roles`)

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/v1/roles` | 역할 목록 |
| `GET` | `/v1/roles/{id}` | ID로 역할 조회 |
| `GET` | `/v1/roles/name/{name}` | 이름으로 역할 조회 |
| `POST` | `/v1/roles` | 역할 생성 |
| `PUT` | `/v1/roles` | 역할 생성/수정 (upsert) |
| `PUT` | `/v1/roles/restore` | 삭제된 역할 복원 |
| `PATCH` | `/v1/roles/{id}` | JSON Patch 부분 수정 |
| `DELETE` | `/v1/roles/{id}` | 역할 삭제 |

### 5.4 Bot API (`/v1/bots`)

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/v1/bots` | 봇 목록 |
| `GET` | `/v1/bots/{id}` | ID로 봇 조회 |
| `GET` | `/v1/bots/name/{name}` | 이름으로 봇 조회 |
| `POST` | `/v1/bots` | 봇 생성 |
| `PUT` | `/v1/bots` | 봇 생성/수정 (upsert) |
| `PUT` | `/v1/bots/restore` | 삭제된 봇 복원 |
| `PATCH` | `/v1/bots/{id}` | JSON Patch 부분 수정 |
| `DELETE` | `/v1/bots/{id}` | 봇 삭제 |

### 5.5 Policy API (`/v1/policies`)

| Method | Path | 설명 |
|--------|------|------|
| `GET` | `/v1/policies` | 정책 목록 |
| `GET` | `/v1/policies/{id}` | ID로 정책 조회 |
| `GET` | `/v1/policies/name/{name}` | 이름으로 정책 조회 |
| `POST` | `/v1/policies` | 정책 생성 |
| `PUT` | `/v1/policies` | 정책 생성/수정 (upsert) |
| `PUT` | `/v1/policies/restore` | 삭제된 정책 복원 |
| `PATCH` | `/v1/policies/{id}` | JSON Patch 부분 수정 |
| `DELETE` | `/v1/policies/{id}` | 정책 삭제 |

---

## 6. 시드 데이터 (초기 데이터)

시스템 시작 시 `OpenMetadataOperations.java`에서 JSON 시드 파일을 로드하여 자동 생성한다.

> 소스: `openmetadata-service/src/main/resources/json/data/`

### 6.1 생성 순서

```
1. Policy 시드 (15개)  ← json/data/policy/*.json
2. Role 시드 (15개)    ← json/data/role/*.json
3. Organization 팀     ← TeamRepository.initOrganization()
4. Bot 시드 (8개)      ← json/data/bot/*.json
5. Bot User 시드 (8개) ← json/data/botUser/*.json
```

### 6.2 시스템 기본 정책

| 정책 | 설명 |
|------|------|
| `OrganizationPolicy` | 전체 조직 기본 정책 (소유자 전권, 전체 ViewAll) |
| `DataConsumerPolicy` | 데이터 소비자 (EditDescription, EditTags 등) |
| `DataStewardPolicy` | 데이터 스튜어드 (메타데이터 전체 수정) |
| `TeamOnlyPolicy` | 팀 전용 정책 |
| `DomainAccessPolicy` | 도메인 접근 정책 |
| `IngestionBotPolicy` | Ingestion 봇 정책 |
| `ProfilerBotPolicy` | Profiler 봇 정책 |
| 기타 | `DefaultBotPolicy`, `GovernanceBotPolicy`, `LineageBotPolicy` 등 |

**OrganizationPolicy 예시:**
```json
{
  "name": "OrganizationPolicy",
  "displayName": "Organization Policy",
  "description": "Policy for all the users of an organization.",
  "enabled": true,
  "provider": "system",
  "rules": [
    {
      "name": "OrganizationPolicy-Owner-Rule",
      "description": "Allow all the operations on an entity for the owner.",
      "resources": ["all"],
      "operations": ["All"],
      "effect": "allow",
      "condition": "isOwner()"
    },
    {
      "name": "OrganizationPolicy-NoOwner-Rule",
      "resources": ["all"],
      "operations": ["EditOwners"],
      "effect": "allow",
      "condition": "noOwner()"
    },
    {
      "name": "OrganizationPolicy-ViewAll-Rule",
      "resources": ["all"],
      "operations": ["ViewAll"],
      "effect": "allow"
    }
  ]
}
```

### 6.3 시스템 기본 역할

| 역할 | 연결 정책 | 설명 |
|------|-----------|------|
| `DataConsumer` | DataConsumerPolicy | 데이터 자산 소비자 |
| `DataSteward` | DataStewardPolicy | 메타데이터 관리자 |
| `IngestionBotRole` | IngestionBotPolicy | Ingestion 봇 역할 |
| `ProfilerBotRole` | ProfilerBotPolicy | Profiler 봇 역할 |
| `DefaultBotRole` | DefaultBotPolicy | 기본 봇 역할 |
| 기타 | 각 전용 정책 | `GovernanceBotRole`, `LineageBotRole`, `UsageBotRole` 등 |

### 6.4 시스템 기본 봇

| 봇 | 봇 사용자 | 역할 | 용도 |
|----|-----------|------|------|
| `ingestion-bot` | `ingestion-bot` | IngestionBotRole | 메타데이터 수집 |
| `governance-bot` | `governance-bot` | GovernanceBotRole | 거버넌스 워크플로우 |
| `profiler-bot` | `profiler-bot` | ProfilerBotRole | 데이터 프로파일링 |
| `usage-bot` | `usage-bot` | UsageBotRole | 사용량 수집 |
| `lineage-bot` | `lineage-bot` | LineageBotRole | 리니지 수집 |
| `testsuite-bot` | `testsuite-bot` | QualityBotRole | 데이터 품질 테스트 |
| `autoClassification-bot` | `autoClassification-bot` | AutoClassificationBotRole | 자동 분류 |
| `scim-bot` | `scim-bot` | ScimBotRole | SCIM 프로비저닝 |

### 6.5 Organization 팀 (자동 생성)

```java
// TeamRepository.initOrganization() 에서 생성
Team team = new Team()
    .withName("Organization")
    .withDisplayName("Organization")
    .withDescription("Organization under which all the other team hierarchy is created")
    .withTeamType(ORGANIZATION)
    .withUpdatedBy("admin")
    .withPolicies(List.of(organizationPolicy))
    .withDefaultRoles(List.of(dataConsumerRole));
```

---

## 7. 실제 API 호출 예시

> 기본 URL: `http://localhost:8585/api`
> 인증 헤더: `Authorization: Bearer <JWT_TOKEN>`

### 7.1 사용자 생성 (POST)

**Request:**
```bash
curl -X POST 'http://localhost:8585/api/v1/users' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <TOKEN>' \
  -d '{
    "name": "john.doe",
    "email": "john.doe@company.com",
    "displayName": "John Doe",
    "description": "Data Engineer at Analytics team",
    "isAdmin": false,
    "teams": ["<team-uuid>"],
    "roles": ["<role-uuid>"],
    "profile": {
      "images": { "image": "https://image.com/john.png" }
    },
    "timezone": "Asia/Seoul"
  }'
```

**Response (201):**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "john.doe",
  "fullyQualifiedName": "john.doe",
  "email": "john.doe@company.com",
  "displayName": "John Doe",
  "description": "Data Engineer at Analytics team",
  "version": 0.1,
  "updatedAt": 1709913600000,
  "updatedBy": "admin",
  "isBot": false,
  "isAdmin": false,
  "deleted": false,
  "href": "http://localhost:8585/api/v1/users/a1b2c3d4-..."
}
```

### 7.2 사용자 목록 조회 (GET)

```bash
# 전체 목록 (페이지네이션)
curl -X GET 'http://localhost:8585/api/v1/users?limit=10&fields=teams,roles'

# 관리자만
curl -X GET 'http://localhost:8585/api/v1/users?isAdmin=true'

# 특정 팀 소속
curl -X GET 'http://localhost:8585/api/v1/users?team=data-analytics&fields=teams,roles'
```

### 7.3 사용자 상세 조회 (GET)

```bash
# ID로 조회
curl -X GET 'http://localhost:8585/api/v1/users/<uuid>?fields=teams,roles,profile'

# 이름으로 조회
curl -X GET 'http://localhost:8585/api/v1/users/name/john.doe?fields=teams,roles'
```

**Response:**
```json
{
  "id": "a1b2c3d4-...",
  "name": "john.doe",
  "email": "john.doe@company.com",
  "displayName": "John Doe",
  "teams": [
    { "id": "team-uuid-...", "type": "team", "name": "data-analytics", "displayName": "Data Analytics" }
  ],
  "roles": [
    { "id": "role-uuid-...", "type": "role", "name": "DataConsumer", "displayName": "Data Consumer" }
  ],
  "inheritedRoles": [
    { "id": "role-uuid-...", "type": "role", "name": "DataConsumer" }
  ]
}
```

### 7.4 사용자 수정 (PATCH — JSON Patch)

```bash
curl -X PATCH 'http://localhost:8585/api/v1/users/<uuid>' \
  -H 'Content-Type: application/json-patch+json' \
  -H 'Authorization: Bearer <TOKEN>' \
  -d '[
    { "op": "replace", "path": "/displayName", "value": "John D. Doe" },
    { "op": "replace", "path": "/description", "value": "Senior Data Engineer" }
  ]'
```

### 7.5 사용자 수정 (PUT — upsert)

```bash
curl -X PUT 'http://localhost:8585/api/v1/users' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <TOKEN>' \
  -d '{
    "name": "john.doe",
    "email": "john.doe@company.com",
    "displayName": "John D. Doe",
    "description": "Senior Data Engineer",
    "teams": ["<team-uuid>"],
    "roles": ["<role-uuid>"]
  }'
```

### 7.6 사용자 삭제/복원 (DELETE / PUT)

```bash
# Soft delete
curl -X DELETE 'http://localhost:8585/api/v1/users/<uuid>'

# Hard delete
curl -X DELETE 'http://localhost:8585/api/v1/users/<uuid>?hardDelete=true&recursive=true'

# 복원
curl -X PUT 'http://localhost:8585/api/v1/users/restore' \
  -H 'Content-Type: application/json' \
  -d '{ "id": "<uuid>" }'
```

### 7.7 팀 생성 — 계층 구조

```bash
# 1) BusinessUnit 생성
curl -X POST 'http://localhost:8585/api/v1/teams' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <TOKEN>' \
  -d '{
    "name": "engineering",
    "displayName": "Engineering",
    "teamType": "BusinessUnit",
    "description": "Engineering Business Unit"
  }'

# 2) Division 생성 (engineering 하위)
curl -X POST 'http://localhost:8585/api/v1/teams' \
  -d '{
    "name": "backend-division",
    "displayName": "Backend Division",
    "teamType": "Division",
    "parents": ["<engineering-uuid>"]
  }'

# 3) Department 생성
curl -X POST 'http://localhost:8585/api/v1/teams' \
  -d '{
    "name": "platform-dept",
    "displayName": "Platform Department",
    "teamType": "Department",
    "parents": ["<backend-division-uuid>"]
  }'

# 4) Group 생성 (사용자 포함)
curl -X POST 'http://localhost:8585/api/v1/teams' \
  -d '{
    "name": "data-platform",
    "displayName": "Data Platform",
    "teamType": "Group",
    "parents": ["<platform-dept-uuid>"],
    "users": ["<user1-uuid>", "<user2-uuid>"],
    "defaultRoles": ["<role-uuid>"],
    "policies": ["<policy-uuid>"]
  }'
```

### 7.8 팀 계층 조회

```bash
curl -X GET 'http://localhost:8585/api/v1/teams/hierarchy?isJoinable=true'
```

### 7.9 팀 상세 조회

```bash
curl -X GET 'http://localhost:8585/api/v1/teams/name/data-platform?fields=users,defaultRoles,policies,parents,childrenCount,userCount'
```

**Response:**
```json
{
  "id": "team-uuid-...",
  "name": "data-platform",
  "displayName": "Data Platform",
  "teamType": "Group",
  "parents": [
    { "id": "...", "type": "team", "name": "platform-dept" }
  ],
  "users": [
    { "id": "...", "type": "user", "name": "john.doe" },
    { "id": "...", "type": "user", "name": "jane.smith" }
  ],
  "defaultRoles": [
    { "id": "...", "type": "role", "name": "DataConsumer" }
  ],
  "policies": [
    { "id": "...", "type": "policy", "name": "DataConsumerPolicy" }
  ],
  "userCount": 2,
  "childrenCount": 0,
  "isJoinable": true
}
```

### 7.10 팀 사용자 관리

```bash
# 팀 사용자 목록 갱신 (전체 교체)
curl -X PUT 'http://localhost:8585/api/v1/teams/<team-uuid>/users' \
  -H 'Content-Type: application/json' \
  -d '[
    { "id": "<user1-uuid>", "type": "user" },
    { "id": "<user2-uuid>", "type": "user" }
  ]'

# 팀에서 특정 사용자 제거
curl -X DELETE 'http://localhost:8585/api/v1/teams/<team-uuid>/users/<user-uuid>'
```

### 7.11 역할 생성/조회

```bash
# 역할 생성
curl -X POST 'http://localhost:8585/api/v1/roles' \
  -d '{
    "name": "CustomAnalyst",
    "displayName": "Custom Analyst",
    "description": "Custom role for data analysts",
    "policies": ["DataConsumerPolicy"]
  }'

# 역할 조회 (관련 정책/팀/사용자 포함)
curl -X GET 'http://localhost:8585/api/v1/roles/name/DataConsumer?fields=policies,teams,users'
```

### 7.12 인증 관련

```bash
# 로그인
curl -X POST 'http://localhost:8585/api/v1/users/login' \
  -H 'Content-Type: application/json' \
  -d '{ "email": "admin@openmetadata.org", "password": "YWRtaW4=" }'

# 토큰 갱신
curl -X POST 'http://localhost:8585/api/v1/users/refresh' \
  -d '{ "refreshToken": "<refresh-token>" }'

# 비밀번호 변경
curl -X PUT 'http://localhost:8585/api/v1/users/changePassword' \
  -d '{
    "username": "admin",
    "oldPassword": "old-password",
    "newPassword": "new-password",
    "confirmPassword": "new-password"
  }'

# 개인 액세스 토큰 생성
curl -X PUT 'http://localhost:8585/api/v1/users/security/token' \
  -d '{ "tokenName": "my-api-token", "JWTTokenExpiry": "30" }'
```

### 7.13 CSV 가져오기/내보내기

```bash
# 팀 내보내기
curl -X GET 'http://localhost:8585/api/v1/teams/name/data-analytics/export'

# 사용자 내보내기
curl -X GET 'http://localhost:8585/api/v1/users/export'

# 팀 가져오기
curl -X PUT 'http://localhost:8585/api/v1/teams/name/data-analytics/import' \
  -H 'Content-Type: text/plain' \
  -d 'name,displayName,description,teamType,parents,owners,isJoinable,defaultRoles,policies
new-team,New Team,A new team,Group,Organization,,true,DataConsumer,'
```

---

## 8. 데이터 흐름 (End-to-End)

### 8.1 사용자 생성 전체 흐름

```
Client                     Server
  │
  │  POST /v1/users
  │  { "name": "john.doe", "email": "john.doe@company.com", "teams": ["<uuid>"] }
  │─────────────────────────►│
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 1. UserResource.create()                          │
  │                    │    - JWT 토큰 검증 (인증)                            │
  │                    │    - 권한 확인 (인가)                                │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 2. UserMapper.createToEntity()                     │
  │                    │    - name → 소문자 변환 ("john.doe")                 │
  │                    │    - email → 소문자 변환                             │
  │                    │    - teams UUID → EntityReference 검증              │
  │                    │    - roles UUID → EntityReference 검증              │
  │                    │    - User 엔티티 객체 생성                           │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 3. UserRepository.prepare()                        │
  │                    │    - 이메일 중복 체크 (checkEmailAlreadyExists)       │
  │                    │    - teams/roles EntityReference 해석               │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 4. UserRepository.storeEntity()                    │
  │                    │    - UserDAO.insert()                              │
  │                    │    ┌───────────────────────────────────────┐       │
  │                    │    │ INSERT INTO user_entity (json, ...)   │       │
  │                    │    │ VALUES ('{"id":"...","name":"john.doe",│      │
  │                    │    │  "email":"john.doe@company.com",...}') │       │
  │                    │    └───────────────────────────────────────┘       │
  │                    │    * id, name, email 등은 GENERATED 컬럼으로 자동 추출│
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 5. UserRepository.storeRelationships()             │
  │                    │    ┌───────────────────────────────────────┐       │
  │                    │    │ INSERT INTO entity_relationship       │       │
  │                    │    │ (fromid, toid, fromentity, toentity,  │       │
  │                    │    │  relation)                            │       │
  │                    │    │ VALUES                                │       │
  │                    │    │ ('<team-uuid>', '<user-uuid>',        │       │
  │                    │    │  'team', 'user', 10)  -- HAS          │       │
  │                    │    │                                       │       │
  │                    │    │ ('<user-uuid>', '<role-uuid>',        │       │
  │                    │    │  'user', 'role', 10)  -- HAS          │       │
  │                    │    └───────────────────────────────────────┘       │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │◄─────────────────────────│
  │  201 Created
  │  { "id": "...", "name": "john.doe", ... }
```

### 8.2 사용자 조회 전체 흐름

```
Client                     Server
  │
  │  GET /v1/users/name/john.doe?fields=teams,roles,profile
  │─────────────────────────►│
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 1. UserResource.getByName()                        │
  │                    │    - JWT 토큰 검증                                  │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 2. UserRepository.getByName()                      │
  │                    │    ┌─────────────────────────────────────────┐     │
  │                    │    │ SELECT json FROM user_entity             │     │
  │                    │    │ WHERE namehash = hash('john.doe')        │     │
  │                    │    └─────────────────────────────────────────┘     │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │                    ┌─────┴──────────────────────────────────────────────┐
  │                    │ 3. UserRepository.setFields() — fields 파라미터 처리│
  │                    │                                                    │
  │                    │  fields=teams:                                     │
  │                    │    fetchAndSetTeams()                              │
  │                    │    ┌─────────────────────────────────────────┐     │
  │                    │    │ SELECT toid FROM entity_relationship     │     │
  │                    │    │ WHERE fromid='<user-uuid>'              │     │
  │                    │    │   AND fromentity='team'                 │     │
  │                    │    │   AND toentity='user'                   │     │
  │                    │    │   AND relation=10  -- HAS               │     │
  │                    │    └─────────────────────────────────────────┘     │
  │                    │                                                    │
  │                    │  fields=roles:                                     │
  │                    │    fetchAndSetRoles()                              │
  │                    │    ┌─────────────────────────────────────────┐     │
  │                    │    │ SELECT toid FROM entity_relationship     │     │
  │                    │    │ WHERE fromid='<user-uuid>'              │     │
  │                    │    │   AND fromentity='user'                 │     │
  │                    │    │   AND toentity='role'                   │     │
  │                    │    │   AND relation=10  -- HAS               │     │
  │                    │    └─────────────────────────────────────────┘     │
  │                    │                                                    │
  │                    │  각 참조 ID로 team_entity, role_entity에서 상세 조회   │
  │                    │  → EntityReference 객체 조합                        │
  │                    └─────┬──────────────────────────────────────────────┘
  │                          │
  │◄─────────────────────────│
  │  200 OK
  │  { "id": "...", "name": "john.doe", "teams": [...], "roles": [...] }
```

### 8.3 팀 계층 생성 → DB 저장 흐름

```
API 호출 순서:

1. POST /v1/teams  { "name": "engineering", "teamType": "BusinessUnit" }
   └─ team_entity에 저장
   └─ entity_relationship: Organization --PARENT_OF(9)--> engineering

2. POST /v1/teams  { "name": "backend", "teamType": "Division", "parents": ["<eng-uuid>"] }
   └─ team_entity에 저장
   └─ entity_relationship: engineering --PARENT_OF(9)--> backend

3. POST /v1/teams  { "name": "platform", "teamType": "Group", "parents": ["<back-uuid>"], "users": ["<user-uuid>"] }
   └─ team_entity에 저장
   └─ entity_relationship: backend --PARENT_OF(9)--> platform
   └─ entity_relationship: platform --HAS(10)--> user

결과 DB 상태 (entity_relationship):
┌──────────────────┬─────────────────┬────────────┬──────────┬──────────┐
│ fromid           │ toid            │ fromentity │ toentity │ relation │
├──────────────────┼─────────────────┼────────────┼──────────┼──────────┤
│ Organization-uuid│ engineering-uuid│ team       │ team     │ 9        │
│ engineering-uuid │ backend-uuid    │ team       │ team     │ 9        │
│ backend-uuid     │ platform-uuid   │ team       │ team     │ 9        │
│ platform-uuid    │ user-uuid       │ team       │ user     │ 10       │
└──────────────────┴─────────────────┴────────────┴──────────┴──────────┘
```

### 8.4 시스템 초기화 전체 흐름

```
OpenMetadataOperations.java 실행
       │
       ▼
  PolicyRepository.initSeedDataFromResources()
       │  json/data/policy/*.json 15개 로드
       │  → policy_entity 테이블에 INSERT
       │
       ▼
  RoleRepository.initializeEntity()
       │  json/data/role/*.json 15개 로드
       │  → role_entity 테이블에 INSERT
       │  → entity_relationship: role --HAS(10)--> policy
       │
       ▼
  TeamRepository.initOrganization()
       │  Organization 팀 생성
       │  → team_entity에 INSERT (teamType=Organization)
       │  → entity_relationship: Organization --HAS(10)--> OrganizationPolicy
       │  → entity_relationship: Organization --DEFAULTS_TO(20)--> DataConsumer
       │
       ▼
  Bot 초기화 (각 봇마다)
       │  json/data/bot/*.json 8개 로드
       │  json/data/botUser/*.json 8개 로드
       │  → bot_entity 테이블에 INSERT
       │  → user_entity 테이블에 INSERT (isBot=true)
       │  → entity_relationship: bot --CONTAINS(0)--> user
       │  → entity_relationship: user --HAS(10)--> role
       │
       ▼
  시스템 준비 완료
```

---

## 9. 소스 코드 위치 인덱스

### JSON Schema (엔티티 정의)

| 파일 | 용도 |
|------|------|
| `openmetadata-spec/.../entity/teams/user.json` | User 엔티티 스키마 |
| `openmetadata-spec/.../entity/teams/team.json` | Team 엔티티 스키마 |
| `openmetadata-spec/.../api/teams/createUser.json` | CreateUser 요청 DTO 스키마 |
| `openmetadata-spec/.../api/teams/createTeam.json` | CreateTeam 요청 DTO 스키마 |
| `openmetadata-spec/.../api/teams/createRole.json` | CreateRole 요청 DTO 스키마 |
| `openmetadata-spec/.../type/entityRelationship.json` | 관계 유형 enum 정의 |

### Database Schema

| 파일 | 용도 |
|------|------|
| `bootstrap/sql/schema/postgres.sql` | PostgreSQL 전체 테이블 DDL |
| `bootstrap/sql/schema/mysql.sql` | MySQL 전체 테이블 DDL |
| `bootstrap/sql/migrations/native/` | 버전별 마이그레이션 SQL |

### REST API Resource

| 파일 | Base Path |
|------|-----------|
| `openmetadata-service/.../resources/teams/UserResource.java` | `/v1/users` |
| `openmetadata-service/.../resources/teams/TeamResource.java` | `/v1/teams` |
| `openmetadata-service/.../resources/teams/RoleResource.java` | `/v1/roles` |
| `openmetadata-service/.../resources/bots/BotResource.java` | `/v1/bots` |
| `openmetadata-service/.../resources/policies/PolicyResource.java` | `/v1/policies` |

### Mapper (DTO → Entity)

| 파일 | 변환 |
|------|------|
| `openmetadata-service/.../resources/teams/UserMapper.java` | CreateUser → User |
| `openmetadata-service/.../resources/teams/TeamMapper.java` | CreateTeam → Team |
| `openmetadata-service/.../resources/teams/RoleMapper.java` | CreateRole → Role |
| `openmetadata-service/.../resources/bots/BotMapper.java` | CreateBot → Bot |
| `openmetadata-service/.../resources/policies/PolicyMapper.java` | CreatePolicy → Policy |

### Repository (비즈니스 로직)

| 파일 | 테이블 |
|------|--------|
| `openmetadata-service/.../jdbi3/UserRepository.java` | user_entity |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | team_entity |
| `openmetadata-service/.../jdbi3/RoleRepository.java` | role_entity |
| `openmetadata-service/.../jdbi3/BotRepository.java` | bot_entity |
| `openmetadata-service/.../jdbi3/PolicyRepository.java` | policy_entity |
| `openmetadata-service/.../jdbi3/TokenRepository.java` | user_tokens |

### DAO (SQL 인터페이스)

| 파일 | 내부 인터페이스 |
|------|-----------------|
| `openmetadata-service/.../jdbi3/CollectionDAO.java` | UserDAO, TeamDAO, RoleDAO, BotDAO, PolicyDAO, TokenDAO |

### 시드 데이터

| 경로 | 내용 |
|------|------|
| `openmetadata-service/.../json/data/policy/*.json` | 시스템 기본 정책 15개 |
| `openmetadata-service/.../json/data/role/*.json` | 시스템 기본 역할 15개 |
| `openmetadata-service/.../json/data/bot/*.json` | 시스템 봇 8개 |
| `openmetadata-service/.../json/data/botUser/*.json` | 봇 사용자 8개 |

### 초기화 코드

| 파일 | 메서드 |
|------|--------|
| `openmetadata-service/.../util/OpenMetadataOperations.java` | 전체 시드 로딩 오케스트레이션 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | `initOrganization()` |
| `openmetadata-service/.../Entity.java` | `ORGANIZATION_NAME`, `ADMIN_USER_NAME` 상수 |

### 통합 테스트

| 파일 | 테스트 대상 |
|------|------------|
| `openmetadata-integration-tests/.../UserResourceIT.java` | User CRUD, 인증, 프로필 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | Team CRUD, 계층, 사용자 관리 |
| `openmetadata-integration-tests/.../factories/UserTestFactory.java` | 테스트 사용자 생성 유틸 |
