---
module: public-api
path: packages/svelte/src
keywords: API, mount, hydrate, lifecycle, context, runes
tags: [api-runtime, api-compiler]
---

# Public API

> [!info] 학습 목표
> Svelte 5의 공개 API --- 개발자가 직접 사용하는 함수와 타입을 이해한다.

**난이도:** ★★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `src/index-client.js` | 클라이언트 측 공개 API 구현 |
| `src/index-server.js` | 서버 측 공개 API 구현 |
| `src/index.d.ts` | TypeScript 타입 정의 |
| `src/internal/client/render.js` | mount/hydrate 구현체 |

## API 카테고리 개요

```
+--------------------------------------------------+
|                svelte (Public API)                |
+--------------------------------------------------+
|                                                   |
|  Lifecycle        Component        Context        |
|  +-----------+   +-----------+   +------------+   |
|  | onMount   |   | mount()   |   | setContext  |   |
|  | onDestroy |   | hydrate() |   | getContext  |   |
|  | before*   |   | unmount() |   | hasContext  |   |
|  | after*    |   |           |   | getAll...  |   |
|  +-----------+   +-----------+   +------------+   |
|                                                   |
|  Effect Control       Runes (컴파일러 매직)       |
|  +-------------+     +----------------------+     |
|  | flushSync() |     | $state    $derived   |     |
|  | tick()      |     | $effect   $props     |     |
|  | untrack()   |     | $bindable $inspect   |     |
|  | fork()      |     | $host               |     |
|  +-------------+     +----------------------+     |
|                                                   |
|  Legacy (Svelte 4 호환)                           |
|  +--------------------------------------------+  |
|  | beforeUpdate | afterUpdate | createEvent...|  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
```

## Lifecycle Hooks

### onMount

컴포넌트가 DOM에 마운트된 직후 한 번 실행된다. 반환된 함수는 언마운트 시 클린업으로 호출된다.

```js
// 내부 구현 (index-client.js)
export function onMount(fn) {
  // component_context 확인 -> user_effect로 래핑
  user_effect(() => {
    const cleanup = untrack(fn);
    if (typeof cleanup === 'function') return cleanup;
  });
}
```

> [!warning] 주의
> `onMount`는 SSR에서 실행되지 않는다. 서버 측 `index-server.js`에서는 noop으로 처리된다.

### onDestroy

컴포넌트가 언마운트되기 직전에 실행된다. 내부적으로 `onMount(() => () => fn())`으로 구현되어 있다.

### beforeUpdate / afterUpdate (Legacy)

Svelte 4 호환을 위해 유지되는 레거시 훅. `legacy_mode_flag`가 활성화된 경우에만 동작한다.

## Component Mounting

| 함수 | 설명 |
|------|------|
| `mount(component, options)` | 컴포넌트를 DOM에 마운트 |
| `hydrate(component, options)` | SSR HTML 위에 클라이언트 하이드레이션 |
| `unmount(component)` | 컴포넌트를 DOM에서 제거 |

## Context API

| 함수 | 설명 |
|------|------|
| `setContext(key, value)` | 현재 컴포넌트에 context 값 설정 |
| `getContext(key)` | 부모 컴포넌트 체인에서 context 값 조회 |
| `hasContext(key)` | context 존재 여부 확인 |
| `getAllContexts()` | 모든 context를 Map으로 반환 |

## Effect Control

| 함수 | 설명 |
|------|------|
| `flushSync()` | 대기 중인 반응성 업데이트를 동기적으로 즉시 실행 |
| `tick()` | 다음 마이크로태스크까지 대기 (Promise 기반) |
| `untrack(fn)` | fn 내부의 반응성 추적을 비활성화 |
| `fork()` | 반응성 컨텍스트를 포크하여 독립적 실행 |
| `getAbortSignal()` | derived/effect 재실행 시 자동 취소되는 AbortSignal 반환 |

## Runes (컴파일러 매직)

Runes는 런타임 함수가 아니라 **컴파일러가 변환하는 특수 구문**이다. DEV 모드에서는 `globalThis`에 에러 핸들러가 등록되어, Svelte 파일 외부에서 사용 시 명확한 에러를 제공한다.

| Rune | 설명 |
|------|------|
| `$state` | 반응성 상태 선언 |
| `$derived` | 파생 상태 (의존성 자동 추적) |
| `$effect` | 부수 효과 (의존성 변경 시 재실행) |
| `$props` | 컴포넌트 props 선언 |
| `$bindable` | 양방향 바인딩 가능한 prop |
| `$inspect` | 디버깅용 값 추적 (DEV only) |
| `$host` | 커스텀 엘리먼트의 host 참조 |

> [!note] DEV 모드 Rune 보호
> `index-client.js` 상단에서 `throw_rune_error`를 통해 `$state`, `$effect` 등을 `globalThis`에 정의하여, `.svelte` 파일 외부에서 사용 시 친절한 에러 메시지를 보여준다.

## Client vs Server 차이

| 기능 | Client (`index-client.js`) | Server (`index-server.js`) |
|------|---------------------------|---------------------------|
| `onMount` | user_effect로 실행 | noop (실행 안 함) |
| `onDestroy` | onMount 기반 클린업 | 동기 실행 |
| `mount/hydrate` | DOM 조작 수행 | 사용 불가 |
| `beforeUpdate` | legacy 모드에서 실행 | noop |
| Runes | 컴파일러가 변환 | 컴파일러가 변환 |
| `createEventDispatcher` | CustomEvent 생성 | 서버용 구현 |

## 테스팅
- 실행: `pnpm test runtime-runes`, `pnpm test store`
- 패턴: 단위/통합 테스트 (Vitest)
- 테스트 위치: `packages/svelte/tests/runtime-runes/`, `packages/svelte/tests/store/`

## Related Notes

- [[Store-API]] --- Store 기반 상태 관리
- [[Motion-API]] --- 애니메이션 및 트랜지션
- [[Practice-Public-API]] --- 연습문제
- [[Reactivity-System]] --- 반응성 시스템 내부 구현
- [[Client-Runtime]] --- 클라이언트 런타임
- [[Server-Runtime]] --- 서버 런타임
