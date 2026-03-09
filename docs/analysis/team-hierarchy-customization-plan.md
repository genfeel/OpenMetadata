# OpenMetadata 팀 계층 구조 커스터마이징 방안 분석

## 1. 현재 구조 및 제약 분석

### 1.1 현재 계층 모델

OpenMetadata는 고정 5단계 팀 계층을 사용한다:

```
Organization → BusinessUnit → Division → Department → Group
```

각 타입별 부모-자식 관계 규칙:

| TeamType | 허용되는 부모 타입 | 허용되는 자식 타입 | 부모 수 제한 |
|----------|-------------------|-------------------|-------------|
| Organization | 없음 (루트) | BU, Division, Dept, Group | 0 (부모 불가) |
| BusinessUnit | BU, Organization | BU, Division, Dept, Group | 1 (단일 부모) |
| Division | Division, BU, Organization | Division, Dept, Group | 1 (단일 부모) |
| Department | Dept, Division, BU, Organization | Dept, Group | 다수 허용 |
| Group | Dept, Division, BU, Organization | 없음 (리프 노드) | 다수 허용 |

### 1.2 계층이 강제되는 코드 포인트 전체 목록

#### 1.2.1 스키마 정의

| 파일 | 라인 | 내용 |
|------|------|------|
| `openmetadata-spec/.../entity/teams/team.json` | 10-21 | `teamType` enum 정의: `["Group","Department","Division","BusinessUnit","Organization"]` |
| `openmetadata-spec/.../entity/teams/team.json` | 5 | 스키마 description에 계층 순서 명시 |
| `openmetadata-spec/.../entity/teams/team.json` | 79-86 | `parents`/`children` 필드의 description에 타입별 허용 관계 명시 |
| `openmetadata-spec/.../entity/teams/teamHierarchy.json` | 5 | `"Organization -> BusinessUnit -> Division -> Department -> Group"` 고정 계층 문서화 |
| `openmetadata-spec/.../entity/teams/teamHierarchy.json` | 47 | `children` description에 타입별 허용 자식 관계 명시 |
| `openmetadata-spec/.../api/teams/createTeam.json` | 38-53 | Create API의 `parents`/`children` description에 동일 규칙 반복 |

#### 1.2.2 백엔드 검증 (Java)

| 파일 | 라인 | 내용 |
|------|------|------|
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 22-26 | `TeamType` static import (BUSINESS_UNIT, DEPARTMENT, DIVISION, GROUP, ORGANIZATION) |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 72 | `TeamType` import |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 417-425 | `prepare()` 메서드: `populateParents()`, `populateChildren()`, `validateHierarchy()` 호출 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 892-915 | `populateChildren()`: switch문으로 각 타입별 허용 자식 타입 검증 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 917-946 | `populateParents()`: switch문으로 각 타입별 허용 부모 타입 검증 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 972-980 | `validateParents()`: 허용 타입 리스트 기반 부모 검증 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 983-990 | `validateChildren()`: 허용 타입 리스트 기반 자식 검증 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1003-1007 | `validateSingleParent()`: BU/Division 단일 부모 제약 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 992-1001 | `validateDefaultPersona()`: Group 타입만 persona 허용 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1018-1021 | `updateTeamUsers()`: Group 타입만 사용자 직접 추가 허용 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1084-1086 | `bulkAddUsers()`: Group 타입만 사용자 벌크 추가 허용 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1130 | Organization 초기화 시 `ORGANIZATION` 타입 하드코딩 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1162-1168 | CSV import: `TeamType.fromValue()` 로 고정 enum 파싱 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1215 | CSV export: `entity.getTeamType().value()` 출력 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1300-1309 | `TeamUpdater.entitySpecificUpdate()`: Group 타입 변경 제약 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1347 | 업데이트 시 persona는 Group만 허용 |
| `openmetadata-service/.../jdbi3/TeamRepository.java` | 1461-1517 | `validateHierarchy()` / `checkCircularReference()`: 순환 참조 검증 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 56 | `INVALID_GROUP_TEAM_UPDATE` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 57-58 | `INVALID_GROUP_TEAM_CHILDREN_UPDATE` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 63-64 | `UNEXPECTED_PARENT` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 65 | `DELETE_ORGANIZATION` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 66-67 | `CREATE_ORGANIZATION` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 68-69 | `CREATE_GROUP` 상수 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java` | 288-312 | `invalidParent()`, `invalidChild()`, `invalidParentCount()`, `invalidTeamOwner()`, `invalidTeamUpdateUsers()` 에러 메시지 생성 메서드 |

#### 1.2.3 프론트엔드 (TypeScript/React)

| 파일 | 라인 | 내용 |
|------|------|------|
| `openmetadata-ui/.../generated/entity/teams/team.ts` | 368-374 | `TeamType` enum (코드 생성됨) |
| `openmetadata-ui/.../generated/entity/teams/teamHierarchy.ts` | 115-121 | `TeamType` enum 중복 (코드 생성됨) |
| `openmetadata-ui/.../generated/api/teams/createTeam.ts` | 236-242 | `TeamType` enum (코드 생성됨) |
| `openmetadata-ui/.../utils/TeamUtils.ts` | 52-70 | `getTeamOptionsFromType()`: 부모 타입별 허용 자식 타입 switch문 |
| `openmetadata-ui/.../utils/TeamUtils.ts` | 79-88 | `isDropRestricted()`: 드래그 앤 드롭 제약 (Group 자식 불가, Division→BU 불가, Dept→BU/Division 불가) |
| `openmetadata-ui/.../components/common/TeamTypeSelect/TeamTypeSelect.component.tsx` | 42-51 | `useMemo`: 부모 타입에 따라 선택 가능 타입 필터링 |
| `openmetadata-ui/.../pages/TeamsPage/AddTeamForm.tsx` | 57-62 | 팀 생성 폼에서 `getTeamOptionsFromType()` 사용 |
| `openmetadata-ui/.../pages/TeamsPage/AddTeamForm.tsx` | 170-172 | 기본값: `teamType: TeamType.Group` |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamsHeaderSection/TeamsInfo.component.tsx` | 312-318 | `TeamTypeSelect` 렌더링 시 `parentTeamType`, `showGroupOption` 전달 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamDetailsV1.utils.tsx` | 18-66 | `getTabs()`: `isGroupType`/`isOrganization` 기준으로 탭 구성 분기 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamHierarchy.tsx` | 38 | `isDropRestricted()` import 및 드래그 앤 드롭 검증 |

#### 1.2.4 테스트

| 파일 | 라인 | 내용 |
|------|------|------|
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 183-200 | `test_createTeamWithDifferentTypes`: 4개 타입 생성 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 203 | `test_createOrganizationNotAllowed`: Organization 생성 불가 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 367-409 | `test_teamHierarchy_parentChild`: Org→BU→Division→Dept 계층 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 412-432 | `test_invalidHierarchy_groupCannotHaveChildren`: Group 자식 제약 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 435 | `test_invalidHierarchy_departmentCannotBeParentOfDivision`: 계층 위반 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 893 | `test_departmentCanHaveMultipleParents`: Department 다중 부모 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 933 | `test_businessUnitCanOnlyHaveOneParent`: BU 단일 부모 제약 테스트 |
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 1075-1108 | CSV import 순환 참조 검증 테스트 |
| `openmetadata-integration-tests/.../TeamDefaultPersonaIT.java` | 210-242 | Group만 persona 허용 테스트 |
| `openmetadata-ui/.../utils/TeamUtils.test.ts` | 17-112 | `isDropRestricted()` 단위 테스트 |
| `openmetadata-ui/.../components/common/TeamTypeSelect/TeamTypeSelect.test.tsx` | 20-57 | `TeamTypeSelect` 컴포넌트 테스트 |
| `openmetadata-ui/.../playwright/e2e/Features/TeamsHierarchy.spec.ts` | 41-100 | E2E 계층 생성 테스트 |
| `openmetadata-ui/.../playwright/e2e/Features/TeamsDragAndDrop.spec.ts` | 47-52 | 드래그 앤 드롭 계층 제약 E2E 테스트 |
| `openmetadata-ui/.../playwright/utils/team.ts` | 136-166 | `getTeamType()`: 테스트 유틸에서 계층 규칙 하드코딩 |
| `ingestion/tests/unit/sdk/test_team_entity.py` | 29-206 | Python SDK 팀 엔티티 테스트 |

#### 1.2.5 데이터베이스

| 파일 | 라인 | 내용 |
|------|------|------|
| `bootstrap/sql/schema/mysql.sql` | ~950 | `teamType varchar(64)` 컬럼: JSON `$.teamType` 추출 (제약 없음, 문자열 저장) |
| `bootstrap/sql/schema/postgres.sql` | ~893 | `teamtype varchar(64)` 컬럼: JSON `->>'teamType'` 추출 (제약 없음, 문자열 저장) |

> **참고**: DB 스키마에는 enum 제약이 없다. `teamType`은 `varchar(64)`로 저장되며 JSON 문서에서 추출되는 가상/생성 컬럼이다. 새로운 타입 추가 시 DB 마이그레이션이 필요하지 않다.

---

## 2. 방안 A: 새로운 TeamType 추가 + 제약 완화

### 2.1 개요

기존 enum에 새로운 타입(예: `SubDivision`, `Section`, `Unit`)을 추가하고, 부모-자식 관계의 switch문을 확장하여 유연한 경로를 허용한다.

**핵심 아이디어**: BU → Dept 직접 연결 등 기존 고정 경로를 우회할 수 있도록 switch문의 허용 타입을 확대한다.

### 2.2 수정 범위

#### 2.2.1 스키마 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-spec/.../entity/teams/team.json:10-21` | `teamType` enum에 새 값 추가 (예: `"SubDivision"`, `"Section"`, `"Unit"`) |
| `openmetadata-spec/.../entity/teams/team.json:5` | 스키마 description 업데이트 |
| `openmetadata-spec/.../entity/teams/team.json:79-86` | `parents`/`children` description 업데이트 |
| `openmetadata-spec/.../entity/teams/teamHierarchy.json:5,47` | 계층 description 업데이트 |
| `openmetadata-spec/.../api/teams/createTeam.json:38-53` | `parents`/`children` description 업데이트 |

#### 2.2.2 백엔드 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-service/.../jdbi3/TeamRepository.java:22-26` | 새 TeamType static import 추가 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:892-915` | `populateChildren()` switch문에 새 타입 case 추가 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:917-946` | `populateParents()` switch문에 새 타입 case 추가 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:1300-1309` | `TeamUpdater` 타입 변경 제약 로직 업데이트 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java:288-312` | 에러 메시지 업데이트 (필요시) |

#### 2.2.3 프론트엔드 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-ui/.../utils/TeamUtils.ts:52-70` | `getTeamOptionsFromType()` switch문에 새 타입 case 추가 |
| `openmetadata-ui/.../utils/TeamUtils.ts:79-88` | `isDropRestricted()` 새 타입 드래그 앤 드롭 규칙 추가 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamDetailsV1.utils.tsx:18-66` | `getTabs()` 새 타입에 대한 탭 구성 추가 |

#### 2.2.4 코드 생성 (자동)

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-ui/.../generated/entity/teams/team.ts` | 스키마에서 자동 생성 (빌드 시) |
| `openmetadata-ui/.../generated/entity/teams/teamHierarchy.ts` | 스키마에서 자동 생성 (빌드 시) |
| `openmetadata-ui/.../generated/api/teams/createTeam.ts` | 스키마에서 자동 생성 (빌드 시) |
| Java 생성 클래스 (`CreateTeam.TeamType`) | `mvn clean install` 시 자동 생성 |

#### 2.2.5 테스트 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 새 타입 생성/계층 통합 테스트 추가, 기존 invalid hierarchy 테스트 업데이트 |
| `openmetadata-ui/.../utils/TeamUtils.test.ts` | `isDropRestricted()` 새 타입 케이스 추가 |
| `openmetadata-ui/.../playwright/utils/team.ts:136-166` | `getTeamType()` 새 타입 매핑 추가 |
| `openmetadata-ui/.../playwright/e2e/Features/TeamsHierarchy.spec.ts` | 새 계층 경로 E2E 시나리오 추가 |

#### 2.2.6 DB 마이그레이션

DB 마이그레이션 불필요 — `teamType`이 `varchar(64)`이므로 새 값이 자동으로 저장된다.

### 2.3 구체적 수정 예시

**예: BU → Department 직접 연결 허용**

`TeamRepository.java` `populateParents()` 수정:

```java
// 기존
case DEPARTMENT:
    validateParents(team, parents, DEPARTMENT, DIVISION, BUSINESS_UNIT, ORGANIZATION);
    break;

// 수정 불필요 — 이미 BUSINESS_UNIT이 Department의 허용 부모에 포함되어 있음
```

> 실제로 현재 코드에서 Department의 부모로 BU가 이미 허용되어 있다. 제약은 주로 "중간 레벨을 건너뛸 수 없다"는 것이 아닌, "특정 타입 조합만 허용"하는 방식이다.

**예: SubDivision 타입 추가**

```java
// populateChildren() 추가
case SUB_DIVISION:
    validateChildren(team, children, DEPARTMENT, GROUP, SUB_DIVISION);
    break;

// populateParents() 추가
case SUB_DIVISION:
    validateSingleParent(team, parentRefs);
    validateParents(team, parents, DIVISION, SUB_DIVISION, BUSINESS_UNIT, ORGANIZATION);
    break;
```

### 2.4 장점

- 기존 아키텍처와 일관성 유지
- 변경 범위가 예측 가능하고 제한적
- 기존 데이터와 완전 호환
- 코드 리뷰가 비교적 용이

### 2.5 단점

- 새 타입이 필요할 때마다 코드 수정 + 빌드 + 배포 필요
- switch문이 점점 비대해짐
- 타입 수가 증가하면 부모-자식 관계 조합이 기하급수적으로 복잡해짐
- 사용자가 자체적으로 타입을 정의할 수 없음

---

## 3. 방안 B: Generic Hierarchy (Custom 타입 + hierarchyLevel)

### 3.1 개요

고정 enum을 폐지하고, 사용자 정의 가능한 팀 타입 시스템을 도입한다. 각 타입에 `hierarchyLevel` (정수)을 부여하고, **자식의 level > 부모의 level** 규칙만으로 계층을 검증한다.

**핵심 아이디어**: 타입 이름과 허용 관계를 코드가 아닌 설정/데이터로 관리한다.

### 3.2 설계

#### 3.2.1 새로운 스키마

```json
{
  "teamType": {
    "description": "Team type name. Built-in: Organization(0), BusinessUnit(1), Division(2), Department(3), Group(4). Custom types can be added.",
    "type": "string"
  },
  "hierarchyLevel": {
    "description": "Hierarchy level for ordering. Lower values are closer to the root. Parent must have a lower level than child.",
    "type": "integer",
    "minimum": 0,
    "default": 4
  }
}
```

#### 3.2.2 검증 규칙 변경

기존 switch문 기반 검증을 다음 규칙으로 대체:

1. **레벨 규칙**: `parent.hierarchyLevel < child.hierarchyLevel`
2. **리프 규칙**: 최고 레벨(예: Group, level=4)은 팀 자식을 가질 수 없음 (isLeaf 속성 또는 특정 레벨 이상)
3. **루트 규칙**: 최저 레벨(level=0)은 부모를 가질 수 없음
4. **단일 부모 규칙**: 설정 가능 (`singleParent: true/false`)

#### 3.2.3 팀 타입 레지스트리

설정 파일이나 DB 테이블로 관리:

```yaml
# openmetadata.yaml 또는 DB team_type_config 테이블
teamTypes:
  - name: Organization
    level: 0
    singleParent: false
    isRoot: true
    canHaveUsers: false
  - name: BusinessUnit
    level: 1
    singleParent: true
    canHaveUsers: false
  - name: Division
    level: 2
    singleParent: true
    canHaveUsers: false
  - name: SubDivision     # 사용자 커스텀
    level: 3
    singleParent: true
    canHaveUsers: false
  - name: Department
    level: 4
    singleParent: false
    canHaveUsers: false
  - name: Section          # 사용자 커스텀
    level: 5
    singleParent: false
    canHaveUsers: false
  - name: Group
    level: 10
    singleParent: false
    isLeaf: true
    canHaveUsers: true
```

### 3.3 수정 범위

#### 3.3.1 스키마 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-spec/.../entity/teams/team.json:10-21` | `teamType`을 enum에서 plain string으로 변경, `hierarchyLevel` 속성 추가 |
| `openmetadata-spec/.../entity/teams/team.json:79-86` | `parents`/`children` description을 레벨 기반 규칙으로 변경 |
| `openmetadata-spec/.../entity/teams/teamHierarchy.json:5,21-24,47` | 계층 description을 레벨 기반으로 변경 |
| `openmetadata-spec/.../api/teams/createTeam.json:11-13` | `teamType`을 plain string으로, `hierarchyLevel` 추가 |
| `openmetadata-spec/.../api/teams/createTeam.json:38-53` | `parents`/`children` description 업데이트 |
| (신규) `openmetadata-spec/.../configuration/teamTypeConfig.json` | 팀 타입 레지스트리 스키마 정의 |

#### 3.3.2 백엔드 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-service/.../jdbi3/TeamRepository.java:22-26,72` | 고정 enum import 제거, 레지스트리 서비스 주입 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:892-915` | `populateChildren()`: switch문을 레벨 기반 비교로 대체 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:917-946` | `populateParents()`: switch문을 레벨 기반 비교로 대체 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:972-1007` | `validateParents()`, `validateChildren()`, `validateSingleParent()`: 레벨 기반 로직으로 재작성 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:992-1001` | `validateDefaultPersona()`: isLeaf 속성 기반으로 변경 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:1018-1021` | `updateTeamUsers()`: isLeaf/canHaveUsers 속성 기반 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:1130` | Organization 초기화 로직 업데이트 |
| `openmetadata-service/.../jdbi3/TeamRepository.java:1162-1168` | CSV import: 문자열 기반 타입 파싱 (enum.fromValue 제거) |
| `openmetadata-service/.../jdbi3/TeamRepository.java:1300-1309` | `TeamUpdater`: isLeaf 기반 타입 변경 제약 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java:56-69` | 에러 메시지를 동적 타입명 기반으로 업데이트 |
| `openmetadata-service/.../exception/CatalogExceptionMessage.java:288-312` | 에러 메시지 메서드: TeamType enum → String 파라미터 |
| (신규) `openmetadata-service/.../service/TeamTypeRegistryService.java` | 팀 타입 레지스트리 관리 서비스 |
| (신규 또는 수정) `conf/openmetadata.yaml` | 팀 타입 설정 섹션 추가 |

#### 3.3.3 프론트엔드 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-ui/.../utils/TeamUtils.ts:52-70` | `getTeamOptionsFromType()`: 동적으로 레지스트리에서 허용 타입 계산 |
| `openmetadata-ui/.../utils/TeamUtils.ts:79-88` | `isDropRestricted()`: hierarchyLevel 비교 기반으로 변경 |
| `openmetadata-ui/.../components/common/TeamTypeSelect/TeamTypeSelect.component.tsx:42-51` | 동적 옵션 계산을 API 기반으로 변경 |
| `openmetadata-ui/.../pages/TeamsPage/AddTeamForm.tsx:57-62,170-172` | 동적 타입 옵션, 기본값 로직 변경 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamsHeaderSection/TeamsInfo.component.tsx:312-318` | 동적 옵션 전달 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamDetailsV1.utils.tsx:18-66` | `getTabs()`: isLeaf/canHaveUsers 속성 기반 분기 |
| `openmetadata-ui/.../components/Settings/Team/TeamDetails/TeamHierarchy.tsx:38` | 동적 드래그 앤 드롭 검증 |
| (신규) 팀 타입 관리 UI (Admin 설정 페이지) | 사용자가 커스텀 타입을 추가/편집할 수 있는 관리 화면 |

#### 3.3.4 코드 생성

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-ui/.../generated/entity/teams/team.ts` | TeamType이 string 타입으로 변경됨 (enum 제거) |
| `openmetadata-ui/.../generated/entity/teams/teamHierarchy.ts` | 동일 |
| `openmetadata-ui/.../generated/api/teams/createTeam.ts` | 동일 |
| Java 생성 클래스 | TeamType이 String으로 변경됨 |

#### 3.3.5 테스트 수정

| 파일 | 수정 내용 |
|------|----------|
| `openmetadata-integration-tests/.../TeamResourceIT.java` | 모든 계층 테스트를 레벨 기반으로 재작성, 커스텀 타입 생성 테스트 추가 |
| `openmetadata-integration-tests/.../TeamDefaultPersonaIT.java` | isLeaf 기반으로 persona 테스트 업데이트 |
| `openmetadata-ui/.../utils/TeamUtils.test.ts` | 레벨 기반 검증 테스트로 재작성 |
| `openmetadata-ui/.../components/common/TeamTypeSelect/TeamTypeSelect.test.tsx` | 동적 옵션 테스트 |
| `openmetadata-ui/.../playwright/e2e/Features/TeamsHierarchy.spec.ts` | 커스텀 타입 E2E 시나리오 추가 |
| `openmetadata-ui/.../playwright/utils/team.ts:136-166` | `getTeamType()` 레벨 기반으로 재작성 |
| `ingestion/tests/unit/sdk/test_team_entity.py` | TeamType 검증 업데이트 |

#### 3.3.6 DB 마이그레이션

| 파일 | 수정 내용 |
|------|----------|
| `bootstrap/sql/migrations/native/...` | (선택) team_type_config 테이블 추가 마이그레이션, 기본 5개 타입 시드 데이터 |
| 기존 `team_entity` 테이블 | 마이그레이션 불필요 — `teamType`이 이미 varchar(64) |

### 3.4 장점

- 코드 수정 없이 새 타입 추가 가능
- 조직별 완전히 다른 계층 구조 지원
- BU → Dept 직접 연결 등 유연한 경로 자연스럽게 지원
- 레벨 기반 검증으로 로직이 단순해짐

### 3.5 단점

- 대규모 변경: 스키마, 백엔드, 프론트엔드 전면 수정
- 기존 enum 기반 코드와의 하위 호환성 깨짐 (마이그레이션 필요)
- API 변경으로 인한 외부 연동 영향
- 레벨이 같은 타입 간 관계 처리 복잡 (Division 아래 Division 같은 재귀 관계)
- 추가적인 관리 UI 개발 필요

---

## 4. 방안별 수정 대상 파일 전체 목록

### 방안 A 수정 파일 (총 ~18개 + 자동 생성 파일)

**스키마 (3개)**
1. `openmetadata-spec/src/main/resources/json/schema/entity/teams/team.json`
2. `openmetadata-spec/src/main/resources/json/schema/entity/teams/teamHierarchy.json`
3. `openmetadata-spec/src/main/resources/json/schema/api/teams/createTeam.json`

**백엔드 (2개)**
4. `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TeamRepository.java`
5. `openmetadata-service/src/main/java/org/openmetadata/service/exception/CatalogExceptionMessage.java`

**프론트엔드 (4개)**
6. `openmetadata-ui/src/main/resources/ui/src/utils/TeamUtils.ts`
7. `openmetadata-ui/src/main/resources/ui/src/components/common/TeamTypeSelect/TeamTypeSelect.component.tsx`
8. `openmetadata-ui/src/main/resources/ui/src/components/Settings/Team/TeamDetails/TeamDetailsV1.utils.tsx`
9. `openmetadata-ui/src/main/resources/ui/src/components/Settings/Team/TeamDetails/TeamHierarchy.tsx`

**자동 생성 (4개, 빌드 시)**
10. `openmetadata-ui/src/main/resources/ui/src/generated/entity/teams/team.ts`
11. `openmetadata-ui/src/main/resources/ui/src/generated/entity/teams/teamHierarchy.ts`
12. `openmetadata-ui/src/main/resources/ui/src/generated/api/teams/createTeam.ts`
13. Java 생성 클래스 (CreateTeam.TeamType enum)

**테스트 (5개)**
14. `openmetadata-integration-tests/src/test/java/org/openmetadata/it/tests/TeamResourceIT.java`
15. `openmetadata-ui/src/main/resources/ui/src/utils/TeamUtils.test.ts`
16. `openmetadata-ui/src/main/resources/ui/playwright/utils/team.ts`
17. `openmetadata-ui/src/main/resources/ui/playwright/e2e/Features/TeamsHierarchy.spec.ts`
18. `openmetadata-ui/src/main/resources/ui/playwright/e2e/Features/TeamsDragAndDrop.spec.ts`

**DB 마이그레이션**: 불필요

---

### 방안 B 수정 파일 (총 ~25개 + 신규 파일 + 자동 생성 파일)

**스키마 (3개 수정 + 1개 신규)**
1. `openmetadata-spec/src/main/resources/json/schema/entity/teams/team.json`
2. `openmetadata-spec/src/main/resources/json/schema/entity/teams/teamHierarchy.json`
3. `openmetadata-spec/src/main/resources/json/schema/api/teams/createTeam.json`
4. **(신규)** `openmetadata-spec/src/main/resources/json/schema/configuration/teamTypeConfig.json`

**백엔드 (2개 수정 + 1~2개 신규)**
5. `openmetadata-service/src/main/java/org/openmetadata/service/jdbi3/TeamRepository.java`
6. `openmetadata-service/src/main/java/org/openmetadata/service/exception/CatalogExceptionMessage.java`
7. **(신규)** `openmetadata-service/src/main/java/org/openmetadata/service/service/TeamTypeRegistryService.java`
8. **(수정)** `conf/openmetadata.yaml` (또는 신규 DB 테이블 + REST API)

**프론트엔드 (6개 수정 + 1~2개 신규)**
9. `openmetadata-ui/src/main/resources/ui/src/utils/TeamUtils.ts`
10. `openmetadata-ui/src/main/resources/ui/src/components/common/TeamTypeSelect/TeamTypeSelect.component.tsx`
11. `openmetadata-ui/src/main/resources/ui/src/components/common/TeamTypeSelect/TeamTypeSelect.interface.ts`
12. `openmetadata-ui/src/main/resources/ui/src/pages/TeamsPage/AddTeamForm.tsx`
13. `openmetadata-ui/src/main/resources/ui/src/components/Settings/Team/TeamDetails/TeamDetailsV1.utils.tsx`
14. `openmetadata-ui/src/main/resources/ui/src/components/Settings/Team/TeamDetails/TeamHierarchy.tsx`
15. **(신규)** 팀 타입 관리 Admin UI 컴포넌트

**자동 생성 (4개, 빌드 시)**
16. `openmetadata-ui/src/main/resources/ui/src/generated/entity/teams/team.ts`
17. `openmetadata-ui/src/main/resources/ui/src/generated/entity/teams/teamHierarchy.ts`
18. `openmetadata-ui/src/main/resources/ui/src/generated/api/teams/createTeam.ts`
19. Java 생성 클래스

**테스트 (7개)**
20. `openmetadata-integration-tests/src/test/java/org/openmetadata/it/tests/TeamResourceIT.java`
21. `openmetadata-integration-tests/src/test/java/org/openmetadata/it/tests/TeamDefaultPersonaIT.java`
22. `openmetadata-ui/src/main/resources/ui/src/utils/TeamUtils.test.ts`
23. `openmetadata-ui/src/main/resources/ui/playwright/utils/team.ts`
24. `openmetadata-ui/src/main/resources/ui/playwright/e2e/Features/TeamsHierarchy.spec.ts`
25. `openmetadata-ui/src/main/resources/ui/playwright/e2e/Features/TeamsDragAndDrop.spec.ts`
26. `ingestion/tests/unit/sdk/test_team_entity.py`

**DB 마이그레이션 (선택)**
27. `bootstrap/sql/migrations/native/.../team_type_config.sql` (팀 타입 레지스트리 테이블)

---

## 5. 비교 분석표

| 항목 | 방안 A: enum 확장 | 방안 B: Generic Hierarchy |
|------|-------------------|--------------------------|
| **수정 파일 수** | ~18개 | ~27개 + 신규 2~4개 |
| **하위 호환성** | 완전 호환 (기존 타입 유지) | 깨짐 (enum → string, API 변경) |
| **기존 데이터 마이그레이션** | 불필요 | hierarchyLevel 값 배정 마이그레이션 필요 |
| **DB 스키마 변경** | 없음 | team_type_config 테이블 추가 (선택) |
| **코드 복잡도 변화** | switch문 case 추가 (점진적 증가) | 초기 복잡하나 이후 안정적 |
| **새 타입 추가 시** | 코드 수정 + 빌드 + 배포 필요 | 설정만 변경 (코드 수정 없음) |
| **유연성** | 제한적 (사전 정의 타입만) | 높음 (사용자 정의 가능) |
| **BU→Dept 직접 연결** | switch문 수정으로 가능 | 레벨 규칙으로 자연스럽게 지원 |
| **재귀 관계 (Div→Div)** | 이미 지원됨 | 동일 레벨 간 관계 처리 필요 |
| **업스트림 기여 가능성** | 높음 (최소 변경) | 낮음 (구조적 변경) |
| **구현 리스크** | 낮음 | 높음 |
| **장기 유지보수** | 타입 증가 시 비용 상승 | 초기 비용 높지만 이후 안정 |
| **외부 연동 영향** | 없음 | API 변경으로 연동 수정 필요 |
| **Python SDK/Ingestion** | 영향 최소 (enum 확장) | 전면 업데이트 필요 |
| **테스트 재작성 범위** | 신규 케이스 추가 | 기존 테스트 대부분 재작성 |

---

## 6. 권장안

### 단기: 방안 A (enum 확장 + 제약 완화)

**이유**:

1. **현재 코드 분석 결과, BU → Dept 직접 연결은 이미 가능하다.** `populateParents()`에서 Department의 허용 부모에 `BUSINESS_UNIT`이 포함되어 있다 (`TeamRepository.java:929-930`). 이는 곧 중간 레벨(Division)을 건너뛰는 것이 backend에서는 이미 허용됨을 의미한다.

2. **프론트엔드가 제약의 주된 병목이다.** `getTeamOptionsFromType()` (`TeamUtils.ts:52-70`)의 switch문이 UI에서 선택 가능한 자식 타입을 제한한다. 이 함수만 수정하면 BU 아래에서 Dept를 직접 생성할 수 있다.

3. **SubDivision/Section 등 추가 레벨이 필요한 경우**, enum에 값을 추가하고 switch문에 case를 추가하는 것만으로 충분하다. 수정 범위가 명확하고 예측 가능하다.

4. **하위 호환성이 유지**되므로 기존 배포, API 연동, 데이터에 영향이 없다.

5. **업스트림 기여 가능성**이 높아 커뮤니티 버전 업그레이드 시 충돌이 최소화된다.

### 장기: 방안 B 고려

조직 구조가 자주 변경되거나, 고객별로 완전히 다른 계층이 필요한 멀티테넌트 환경이라면 방안 B를 고려한다. 단, 방안 B는 API breaking change를 수반하므로 OpenMetadata의 메이저 버전 업그레이드 시점에 맞추는 것이 적절하다.

### 즉시 적용 가능한 최소 변경 (방안 A-lite)

만약 현재 가장 급한 요구가 "BU → Department 직접 연결"이라면, 프론트엔드의 `getTeamOptionsFromType()`만 수정하면 된다:

```typescript
// TeamUtils.ts:52-70 수정
export const getTeamOptionsFromType = (parentType: TeamType) => {
  switch (parentType) {
    case TeamType.Organization:
      return [
        TeamType.BusinessUnit,
        TeamType.Division,
        TeamType.Department,  // 이미 포함
        TeamType.Group,
      ];
    case TeamType.BusinessUnit:
      return [
        TeamType.BusinessUnit,  // BU 아래 BU 추가
        TeamType.Division,
        TeamType.Department,    // BU→Dept 직접 연결 (이미 BE에서 허용)
        TeamType.Group,
      ];
    // ... 나머지 동일
  }
};
```

백엔드 검증 코드는 이미 이 경로를 허용하므로 프론트엔드만 수정하면 즉시 동작한다.
