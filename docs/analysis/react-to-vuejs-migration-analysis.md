# OpenMetadata 프론트엔드 React → Vue.js 마이그레이션 분석

## 1. 프론트엔드 규모 요약

| 항목 | 수량 |
|------|------|
| React 컴포넌트 (`.tsx`) | **1,875개** |
| TypeScript 로직 (`.ts`) | **1,722개** |
| 자동 생성 타입 (`generated/`) | **771개** |
| 스타일 파일 (`.less`) | **417개** |
| Jest 테스트 | **860개** |
| Playwright E2E 테스트 | **385개** |
| 총 코드량 | **~427,000줄** |

### 주요 디렉토리별 파일 수

| 디렉토리 | 파일 수 | 용도 |
|----------|--------|------|
| `components/` | 1,876 | UI 컴포넌트 (70개 하위 디렉토리) |
| `pages/` | 350 | 페이지 레벨 컴포넌트 |
| `hooks/` | 52 | 커스텀 React 훅 |
| `utils/` | 387 | 유틸리티 함수 |
| `rest/` | 75 | API 클라이언트 |
| `constants/` | 89 | 상수 및 설정 |
| `context/` | 21 | React Context 프로바이더 |

---

## 2. React 종속 코드 포인트

### 2.1 React Hooks 사용 빈도

| Hook | 사용 횟수 |
|------|----------|
| `useState` | 2,837 |
| `useMemo` | 2,768 |
| `useCallback` | 2,095 |
| `useEffect` | 1,391 |
| `useRef` | 339 |
| `useContext` | 44 |

### 2.2 React 패턴 사용 현황

| 패턴 | 사용 범위 |
|------|----------|
| `react-router-dom` import | 575개 파일 |
| `useTranslation` (i18n) | 1,640회 |
| Ant Design import | 835개 파일 |
| MUI import | 191개 파일 |
| `React.lazy` 코드 스플리팅 | 47개 파일 |
| `React.forwardRef` | 46개 파일 |
| `React.memo` | 4개 파일 |
| RJSF (React JSON Schema Form) | 72개 파일 |
| `dangerouslySetInnerHTML` | 1개 파일 |

### 2.3 상태 관리

| 항목 | 수량 |
|------|------|
| 커스텀 훅 | 31개 |
| Context Provider | 8개 |
| Zustand 스토어 | 10개 |
| 인터페이스 정의 (`.interface.ts`) | 386개 |

### 2.4 컴포넌트 패턴

- **함수형 컴포넌트**: 1,875개 (100%)
- **클래스 컴포넌트**: 0개

---

## 3. 커스텀 훅 전체 목록 (31개)

Vue.js의 Composable로 변환해야 하는 커스텀 훅:

| 훅 | 용도 |
|----|------|
| `useAbortController` | AbortController 관리 |
| `useCurrentUserStore` | 현재 사용자 상태 (Zustand) |
| `useLocationSearch` | URL 검색 파라미터 |
| `usePaging` | 페이지네이션 |
| `useAlertStore` | 알림 상태 (Zustand) |
| `useApplicationStore` | 앱 설정 (Zustand) |
| `useCanvasEdgeRenderer` | 캔버스 렌더링 |
| `useCanvasMouseEvents` | 캔버스 마우스 이벤트 |
| `useClipBoard` | 클립보드 |
| `useCustomLocation` | 커스텀 라우팅 |
| `useCustomPages` | 커스텀 페이지 |
| `useDomainStore` | 도메인 데이터 (Zustand) |
| `useDownloadProgressStore` | 다운로드 진행률 (Zustand) |
| `useEditableSection` | 편집 가능 섹션 |
| `useElementInView` | Intersection Observer |
| `useEntityRules` | 엔티티 규칙 |
| `useFqn` | FQN 파싱 |
| `useFqnDeepLink` | FQN 딥링크 |
| `useGridEditController` | 그리드 편집 |
| `useGridLayoutDirection` | 레이아웃 방향 (LTR/RTL) |
| `useImage` | 이미지 로딩 |
| `useLineageStore` | 리니지 그래프 (Zustand) |
| `useMapBasedNodesEdges` | 리니지 노드/엣지 |
| `usePubSub` | Pub/Sub 메시징 |
| `useUserProfile` | 사용자 프로필 |
| `useRovingFocus` | 키보드 포커스 관리 |
| `useScrollToElement` | 엘리먼트 스크롤 |
| `useSearchStore` | 검색 상태 (Zustand) |
| `useSidebarItems` | 사이드바 네비게이션 |
| `useTableFilters` | 테이블 필터 |
| `useWelcomeStore` | 웰컴 모달 (Zustand) |

---

## 4. Context Provider 전체 목록 (8개)

Vue.js의 `provide`/`inject` 또는 Pinia 스토어로 변환 필요:

| Provider | 용도 |
|----------|------|
| `AirflowStatusProvider` | Airflow 워크플로우 상태 |
| `AntDConfigProvider` | Ant Design 설정 |
| `AsyncDeleteProvider` | 비동기 삭제 |
| `LineageProvider` | 리니지 그래프 상태 |
| `PermissionProvider` | 사용자 권한/접근 제어 |
| `RuleEnforcementProvider` | 데이터 품질 규칙 |
| `TourProvider` | 가이드 투어 |
| `WebSocketProvider` | WebSocket 연결 |

---

## 5. Zustand 스토어 전체 목록 (10개)

Pinia로 마이그레이션 필요:

| 스토어 | 용도 |
|--------|------|
| `useAlertStore` | 알림 |
| `useApplicationStore` | 앱 설정 |
| `useCurrentUserStore` | 현재 사용자 |
| `useDomainStore` | 도메인 데이터 |
| `useDownloadProgressStore` | 다운로드 진행률 |
| `useLineageStore` | 리니지 그래프 |
| `useSearchStore` | 검색 상태 |
| `useWelcomeStore` | 웰컴 모달 |
| `useGlossary.store` | 용어집 |
| `useTestCase.store` | 테스트 케이스 |

---

## 6. React 전용 라이브러리 → Vue 대체 매핑

### 6.1 대체 라이브러리가 존재하는 경우

| React 라이브러리 | Vue.js 대체 | 영향 파일 수 | 난이도 |
|-----------------|------------|-------------|--------|
| `react` / `react-dom` v18 | `vue` v3 | 전체 | 매우 높음 |
| `react-router-dom` v6 | `vue-router` v4 | 575개 | 높음 |
| `zustand` | `pinia` | 10 스토어 | 중간 |
| `antd` v4 | `ant-design-vue` v4 | 835개 | 매우 높음 |
| `@mui/material` v7 | `vuetify` v3 | 191개 | 높음 |
| `react-hook-form` | `vee-validate` | 중간 | 중간 |
| `@tiptap/react` | `@tiptap/vue-3` | 낮음 | 낮음 |
| `reactflow` | `vue-flow` | 중간 | 중간 |
| `react-dnd` | `vuedraggable` / `vue-dndrop` | 중간 | 중간 |
| `react-i18next` | `vue-i18n` | 1,640회 | 높음 |
| `recharts` | `vue-echarts` 또는 `chart.js` | 중간 | 중간 |
| `react-grid-layout` | `vue-grid-layout` | 낮음 | 낮음 |
| `react-intersection-observer` | `@vueuse/core` `useIntersectionObserver` | 낮음 | 낮음 |
| `react-error-boundary` | Vue `errorCaptured` 훅 | 낮음 | 낮음 |
| `react-helmet-async` | `@unhead/vue` | 낮음 | 낮음 |
| `socket.io-client` | 동일 (프레임워크 무관) | 없음 | 없음 |
| `axios` | 동일 (프레임워크 무관) | 없음 | 없음 |

### 6.2 인증 SDK

| React 인증 라이브러리 | Vue.js 대체 | 비고 |
|---------------------|------------|------|
| `@auth0/auth0-react` | `@auth0/auth0-vue` | API 상이, 전면 재작성 |
| `@azure/msal-react` | `@azure/msal-browser` 직접 래핑 | Vue 전용 SDK 없음 |
| `@okta/okta-react` | `@okta/okta-vue` | API 상이 |
| `react-oidc` | `vue-oidc-client` 또는 직접 구현 | 생태계 불안정 |

### 6.3 대체 불가능 — 직접 구현 필요

| React 라이브러리 | Vue 대체 상태 | 영향 |
|-----------------|-------------|------|
| `@rjsf/core` (React JSON Schema Form) | Vue 동등 라이브러리 없음 | JSON Schema 기반 동적 폼 렌더링 직접 구현 필요 (72개 파일) |
| `react-aria` / `react-aria-components` | Vue 접근성 라이브러리 미성숙 | 접근성 컴포넌트 직접 구현 필요 |
| `@emotion/react` / `styled-components` | Vue scoped styles / CSS modules | 스타일링 패러다임 변경 |

### 6.4 테스트 프레임워크

| React 테스트 도구 | Vue.js 대체 | 영향 |
|------------------|------------|------|
| `@testing-library/react` | `@vue/test-utils` + `@testing-library/vue` | 860개 테스트 전면 재작성 |
| `@testing-library/react-hooks` | Composable 직접 테스트 | 훅 테스트 재작성 |
| `jest` | `vitest` (권장) 또는 jest 유지 | 설정 변경 |
| `@playwright/test` | 동일 (프레임워크 무관) | **재사용 가능** (셀렉터 수정 필요) |

---

## 7. 변환 패턴 대조

### 7.1 컴포넌트 구조

**React (현재)**
```tsx
import { useState, useEffect, useCallback } from 'react';

interface Props {
  title: string;
  onSave: (data: Data) => void;
}

const MyComponent: React.FC<Props> = ({ title, onSave }) => {
  const [loading, setLoading] = useState(false);
  const { t } = useTranslation();

  useEffect(() => {
    fetchData();
  }, []);

  const handleSave = useCallback(() => {
    onSave(data);
  }, [data, onSave]);

  return (
    <div className="my-component">
      <h1>{t('label.title')}</h1>
      {loading ? <Spinner /> : <Content />}
    </div>
  );
};
```

**Vue 3 Composition API (변환 후)**
```vue
<template>
  <div class="my-component">
    <h1>{{ t('label.title') }}</h1>
    <Spinner v-if="loading" />
    <Content v-else />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useI18n } from 'vue-i18n';

interface Props {
  title: string;
}

const props = defineProps<Props>();
const emit = defineEmits<{ save: [data: Data] }>();

const loading = ref(false);
const { t } = useI18n();

onMounted(() => {
  fetchData();
});

const handleSave = () => {
  emit('save', data);
};
</script>
```

### 7.2 주요 변환 매핑

| React 개념 | Vue 3 대응 |
|-----------|-----------|
| `useState` | `ref()` / `reactive()` |
| `useEffect` | `onMounted`, `watch`, `watchEffect` |
| `useCallback` | 일반 함수 (Vue에서 불필요) |
| `useMemo` | `computed()` |
| `useRef` | `ref()` / `templateRef()` |
| `useContext` | `inject()` |
| Context Provider | `provide()` |
| JSX | `<template>` SFC |
| `className` | `class` |
| `{condition && <Comp />}` | `v-if` |
| `{list.map(item => <Comp />)}` | `v-for` |
| `onClick={handler}` | `@click="handler"` |
| `children` prop | `<slot />` |
| `React.lazy()` | `defineAsyncComponent()` |
| `React.forwardRef` | `defineExpose()` |
| `React.memo` | Vue 자체 반응성 (불필요) |
| `dangerouslySetInnerHTML` | `v-html` |
| `.interface.ts` (Props) | `defineProps<T>()` |
| Event handler props | `defineEmits()` |

---

## 8. 빌드 도구 영향

현재 빌드 도구는 **Vite**를 사용 중이며, Vue.js도 Vite를 기본 빌드 도구로 사용하므로 빌드 인프라 변경은 최소화된다.

### 변경 필요

| 항목 | React (현재) | Vue (변환 후) |
|------|-------------|--------------|
| Vite 플러그인 | `@vitejs/plugin-react` | `@vitejs/plugin-vue` |
| SVG 처리 | `vite-plugin-svgr` | `vite-svg-loader` |
| 청크 분할 | `react-vendor` 청크 | `vue-vendor` 청크 |

### 변경 불필요 (재사용 가능)

- Vite 기본 설정
- CSS 코드 스플리팅
- 프로덕션 압축 (gzip + brotli)
- 환경 변수 처리
- 프록시 설정

---

## 9. 재사용 가능 자산

마이그레이션 시 변경 없이 재사용할 수 있는 부분:

| 자산 | 재사용 가능 여부 | 비고 |
|------|----------------|------|
| TypeScript 인터페이스/타입 (`generated/`) | **부분 재사용** | Props 인터페이스 제외한 데이터 타입 재사용 가능 |
| REST API 클라이언트 (`rest/`) | **완전 재사용** | axios 기반, 프레임워크 무관 |
| 유틸리티 함수 (`utils/`) | **대부분 재사용** | React 훅 미사용 유틸만 |
| 상수 (`constants/`) | **완전 재사용** | 프레임워크 무관 |
| LESS/CSS 스타일 | **완전 재사용** | 프레임워크 무관 |
| i18n 번역 파일 (`locales/`) | **완전 재사용** | JSON 포맷, 키 구조 동일 |
| 자산 파일 (SVG, 이미지) | **완전 재사용** | import 방식만 변경 |
| Playwright E2E 테스트 | **부분 재사용** | 셀렉터 수정 필요하나 테스트 로직 유지 |
| Jest 단위 테스트 | **재사용 불가** | `@testing-library/react` 전면 교체 |
| JSON Schema 정의 | **완전 재사용** | 백엔드 스키마, 프레임워크 무관 |

---

## 10. 리스크 분석

### 10.1 고위험

| 리스크 | 설명 |
|--------|------|
| RJSF 대체 부재 | JSON Schema 기반 동적 폼 생성 라이브러리가 Vue 생태계에 없음. 자체 구현 시 핵심 기능(Application 설정, Connection 설정 등)에 심각한 영향 |
| Ant Design 호환성 | `ant-design-vue`는 `antd` React와 API가 유사하지만 완전히 동일하지 않음. 835개 파일에서 컴포넌트별 차이 확인 필요 |
| MUI 대체 | MUI는 React 전용. Vue 대체(`vuetify`)는 API와 디자인 시스템이 완전히 다름. 현재 진행 중인 MUI 마이그레이션이 무효화됨 |
| 테스트 커버리지 상실 | 860개 Jest 테스트 전면 재작성 기간 동안 품질 보증 공백 발생 |
| 인증 흐름 | 4개 인증 프로바이더(Auth0, MSAL, Okta, OIDC) 모두 재통합 필요. 보안 민감 영역 |

### 10.2 중위험

| 리스크 | 설명 |
|--------|------|
| 리니지 그래프 | `reactflow` → `vue-flow` 변환. 복잡한 커스텀 노드/엣지 렌더링 재구현 |
| 에디터 통합 | TipTap Vue 버전 존재하나 커스텀 확장 재작성 필요 |
| 업스트림 동기화 | 마이그레이션 기간 동안 upstream React 코드와의 동기화 불가능. 포크 분기 확대 |
| 학습 곡선 | 팀의 Vue.js 숙련도에 따른 초기 생산성 저하 |

### 10.3 저위험

| 리스크 | 설명 |
|--------|------|
| 라우팅 | `react-router-dom` → `vue-router` 개념적으로 유사, 기계적 변환 가능 |
| 상태 관리 | Zustand → Pinia 패턴이 유사 |
| 빌드 도구 | Vite 유지, 플러그인만 교체 |

---

## 11. 결론

React → Vue.js 마이그레이션은 **사실상 프론트엔드 전체를 재작성하는 수준**이다.

### 핵심 수치

- **변환 대상**: 1,875개 컴포넌트 + 31개 훅 + 10개 스토어 + 8개 프로바이더
- **재작성 대상**: 860개 단위 테스트
- **총 코드량**: ~427,000줄
- **직접 구현 필요**: RJSF (동적 폼), 인증 래퍼, 접근성 컴포넌트
- **재사용 가능**: REST 클라이언트, 상수, 스타일, i18n 파일, Playwright 테스트 (부분)

### 마이그레이션 불필요 — 현재 아키텍처에서 해결 가능한 대안

React를 유지하면서도 달성할 수 있는 개선:

1. **컴포넌트 라이브러리 교체** (Ant Design → MUI): 이미 진행 중
2. **상태 관리 개선**: Zustand는 이미 경량 솔루션
3. **성능 최적화**: React 18의 Concurrent Features 활용
4. **마이크로 프론트엔드**: 특정 모듈만 Vue로 개발하여 점진적 도입 (Module Federation)

마이그레이션의 기술적 이점보다 비용과 리스크가 월등히 크므로, 명확한 비즈니스 요구사항이 없는 한 React 유지를 권장한다.
