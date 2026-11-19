---
module: client-runtime
path: packages/svelte/src/internal/client
keywords: runtime, render, mount, hydrate, component, lifecycle, context
tags: [module-client, arch-runtime]
---

# Client Runtime

> **Purpose**: 브라우저에서 Svelte 컴포넌트를 실행하는 클라이언트 런타임. 컴포넌트의 마운트, 하이드레이션, 반응성 업데이트 실행, 라이프사이클 관리를 담당한다.

> [!important] 중요도: ★★★
> 컴파일된 Svelte 컴포넌트가 브라우저에서 실행될 때 의존하는 핵심 런타임이다.

## Key Files

| 파일 | 역할 | 핵심 export |
|------|------|-------------|
| `runtime.js` | 반응성 실행 엔진 (active_reaction, active_effect 관리) | `get()`, `untrack()`, `active_reaction`, `active_effect` |
| `render.js` | 컴포넌트 마운트/하이드레이트 진입점 | `mount()`, `hydrate()`, `set_text()` |
| `context.js` | 컴포넌트 컨텍스트 (setContext/getContext) | `push()`, `pop()`, `component_context` |
| `proxy.js` | 객체/배열 반응성 Proxy | `proxy()` |
| `loop.js` | RAF 기반 애니메이션 루프 | `loop()` |
| `error-handling.js` | 에러 바운더리 처리 | `handle_error()` |
| `index.js` | 외부 공개 API 모듈 | re-exports |
| `legacy.js` | Svelte 4 호환 레이어 | `captured_signals` |
| `constants.js` | 상태 플래그 상수 | `DIRTY`, `CLEAN`, `STATE_SYMBOL` 등 |
| `validate.js` | 개발 모드 유효성 검사 | 런타임 검증 함수들 |

## Component Lifecycle

```
mount(component, { target })
    |
    v
+--------------------+
| init_operations()  |  DOM 작업 초기화 (lazy)
+--------------------+
    |
    v
+--------------------+
| push(context)      |  컴포넌트 컨텍스트 스택 push
+--------------------+
    |
    v
+--------------------+
| component_root()   |  ROOT_EFFECT 생성
| (effects.js)       |  컴포넌트 함수 실행
+--------------------+
    |
    v
+--------------------+
| 컴포넌트 함수 실행   |  $state, $derived 시그널 생성
|                    |  render_effect로 DOM 바인딩
+--------------------+
    |
    v
+--------------------+
| pop()              |  컴포넌트 컨텍스트 스택 pop
+--------------------+
    |
    v
+--------------------+
| 반응성 업데이트 루프  |  상태 변경 -> Batch -> effect 재실행
+--------------------+
    |
    v  (unmount 시)
+--------------------+
| destroy_effect()   |  effect 트리 정리
| teardown 실행      |  onDestroy, $effect 클린업
+--------------------+
```

## mount() and hydrate()

### mount() (render.js 67행)

클라이언트 전용 렌더링. 빈 DOM 대상에 컴포넌트를 마운트한다.

```javascript
export function mount(component, options) {
    return _mount(component, options);
}
```

`options`에는 `target`, `props`, `context`, `intro`, `events` 등이 포함된다.

### hydrate() (render.js 96행)

SSR로 생성된 HTML을 인터랙티브하게 만든다. 기존 DOM 노드를 재사용하며 이벤트 핸들러와 반응성을 연결한다.

```javascript
export function hydrate(component, options) {
    init_operations();
    options.intro = options.intro ?? false;
    // SSR HTML에서 HYDRATION_START/END 마커를 찾아 노드 매핑
    // ...
}
```

> [!note] mount vs hydrate
> `mount`는 DOM을 새로 생성하고, `hydrate`는 서버에서 렌더링된 기존 DOM을 재활용한다. `hydrate`는 `<!--[-->` / `<!--]-->` 하이드레이션 마커를 사용하여 노드 위치를 추적한다.

## Context API

`context.js`에서 관리한다. `push()`/`pop()`으로 컴포넌트 컨텍스트 스택을 유지한다.

- `component_context`: 현재 실행 중인 컴포넌트의 컨텍스트
- `setContext(key, value)`: 현재 컴포넌트에 컨텍스트 값 저장
- `getContext(key)`: 부모 체인에서 컨텍스트 값 조회

## Error Boundaries

`error-handling.js`의 `handle_error()`가 에러를 처리한다. `<svelte:boundary>` 컴포넌트가 에러 바운더리 역할을 하며, 하위 컴포넌트에서 발생한 에러를 잡아 대체 UI를 표시할 수 있다.

## Dev Mode Features

`dev/` 디렉토리에 개발 전용 기능이 포함되어 있다:

- **tracing.js**: `$inspect.trace()` 구현, 시그널 변경 추적
- **equality.js**: 배열 프로토타입 변경 경고
- **ownership.js**: 소유권 검증 (다른 컴포넌트의 state 변경 감지)
- **hmr.js**: Hot Module Replacement 지원

`DEV` 플래그 (`esm-env`)로 프로덕션 빌드에서 제거된다.

## Key Runtime Variables (runtime.js)

```
active_reaction (73행)  - 현재 실행 중인 reaction (의존성 추적 대상)
active_effect (83행)    - 현재 실행 중인 effect (부모-자식 트리 관리)
untracking (75행)       - untrack() 내부인지 여부
current_sources (95행)  - reaction 내에서 새로 생성된 source들
```

## Dependencies

- [[Reactivity-System]] - 시그널, 이펙트, 배치 시스템
- `#client/constants` - 상태 플래그 상수
- `../shared/utils.js` - 공통 유틸리티

## Used by

- [[DOM-Operations]] - DOM 조작 및 블록 구현
- 컴파일된 컴포넌트 코드 - 런타임 함수 호출
- SvelteKit - `mount()`, `hydrate()` 호출

## 테스팅
- 실행: `pnpm test runtime-runes`, `pnpm test runtime-browser`
- 패턴: 통합 테스트 (Vitest + Playwright)
- 테스트 위치: `packages/svelte/tests/runtime-runes/`, `packages/svelte/tests/runtime-browser/`

## Related Notes

- [[Reactivity-System]] - 반응성의 핵심 메커니즘
- [[DOM-Operations]] - 실제 DOM 조작 레이어
- [[Server-Runtime]] - 서버 측 대응 모듈
- [[Practice-Client-Runtime]] - 연습 문제
