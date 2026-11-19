---
module: reactivity
path: packages/svelte/src/internal/client/reactivity
keywords: signals, effects, derived, state, runes, batch, proxy
tags: [module-reactivity, pattern-signal, pattern-proxy]
---

# Reactivity System

> **Purpose**: Svelte 5의 시그널 기반 반응성 시스템 (Runes). `$state`, `$derived`, `$effect` 등의 Rune이 내부적으로 어떻게 시그널을 생성하고, 의존성을 추적하며, 배치 업데이트를 수행하는지를 담당하는 핵심 모듈이다.

> [!important] 중요도: ★★★
> Svelte 5의 가장 핵심적인 변경사항. 기존의 컴파일러 기반 반응성에서 런타임 시그널 기반으로 전환되었다.

## Key Files

| 파일 | 역할 | 핵심 export |
|------|------|-------------|
| `sources.js` | Source 시그널 생성 및 업데이트 (`$state`) | `source()`, `set()`, `internal_set()`, `state()` |
| `effects.js` | Effect 생성 및 실행 (`$effect`) | `create_effect()`, `user_effect()`, `render_effect()`, `component_root()` |
| `deriveds.js` | Derived 시그널 (`$derived`) | `derived()`, `execute_derived()`, `update_derived()` |
| `batch.js` | 배치 업데이트 스케줄링 | `Batch`, `schedule_effect()`, `flushSync()` |
| `props.js` | 컴포넌트 props 반응성 처리 | `prop()`, `spread_props()` |
| `store.js` | Svelte store 호환 레이어 | `store_get()`, `store_set()` |
| `async.js` | 비동기 반응성 (`async` 모드) | `increment_pending()`, `flatten()` |
| `equality.js` | 값 비교 함수 | `equals()`, `safe_equals()` |
| `status.js` | 시그널 상태 플래그 관리 | `set_signal_status()`, `update_derived_status()` |

## Core Concepts

### Signal (Source)

`source()` 함수가 시그널 객체를 생성한다. `$state`로 선언하면 컴파일러가 이 함수를 호출하도록 변환한다.

```
// 컴파일러 변환 결과
let count = source(0);  // $state(0)에 해당
```

시그널 객체의 구조 (sources.js 77-84행):

```javascript
var signal = {
    f: 0,           // flags (상태 플래그)
    v,              // value (현재 값)
    reactions: null, // 이 시그널을 구독하는 reaction 목록
    equals,         // 값 비교 함수
    rv: 0,          // read version
    wv: 0           // write version
};
```

### Proxy 시스템

객체나 배열에 `$state`를 사용하면 `proxy()` (proxy.js)로 감싸진다. Proxy는 각 프로퍼티마다 개별 source 시그널을 `Map`으로 관리한다 (proxy.js 53행).

```
// proxy.js 핵심 구조
var sources = new Map();          // 프로퍼티별 source 시그널
var version = source(0);          // 전체 객체 버전 추적
var is_proxied_array = is_array(value);  // 배열 특수 처리
```

> [!note] Proxy 대상 제한
> `object_prototype` 또는 `array_prototype`을 프로토타입으로 가진 객체만 프록시된다. 클래스 인스턴스, Map, Set 등은 프록시되지 않는다 (proxy.js 48행).

### Derived

`derived()` 함수는 lazy하게 계산되는 파생 시그널을 생성한다. DERIVED | DIRTY 플래그로 시작하며, 읽힐 때만 계산된다 (deriveds.js 68-95행).

```javascript
const signal = {
    ctx: component_context,
    deps: null,       // 의존하는 시그널 목록
    effects: null,    // 내부에서 생성된 이펙트
    equals,
    f: DERIVED | DIRTY,
    fn,               // 계산 함수
    reactions: null,
    rv: 0,
    v: UNINITIALIZED, // 초기값: 아직 계산 안됨
    wv: 0,
    parent: parent_derived ?? active_effect,
    ac: null          // async controller
};
```

### Effect

`create_effect()` (effects.js 87행)가 모든 이펙트의 기반이다. 부모-자식 관계를 linked list로 관리한다 (`first`, `last`, `prev`, `next`).

이펙트 종류:
- `RENDER_EFFECT`: DOM 업데이트용 (컴파일러가 생성)
- `EFFECT` / `USER_EFFECT`: `$effect()`로 생성
- `EAGER_EFFECT`: `$inspect` 등 즉시 실행
- `BRANCH_EFFECT`: if/each 등 블록 분기
- `BLOCK_EFFECT`: 블록 레벨
- `ROOT_EFFECT`: 컴포넌트 루트

### Batching

여러 상태 변경이 하나의 업데이트로 묶인다. `Batch` 클래스 (batch.js)가 이를 관리한다.

## Reactivity Flow

```
$state 변경 (set)
    |
    v
+-------------------+
| internal_set()    |  값이 다른지 equals()로 비교
+-------------------+
    |
    v
+-------------------+
| mark reactions    |  의존하는 reaction들을 DIRTY로 표시
| as DIRTY          |  (effects, deriveds)
+-------------------+
    |
    v
+-------------------+
| schedule_effect() |  Batch에 effect 등록
+-------------------+
    |
    v
+-------------------+
| Batch.flush()     |  microtask에서 일괄 실행
+-------------------+
    |
    v
+-------------------+
| execute effects   |  DIRTY인 effect 재실행
| (render_effect)   |  derived는 필요시 재계산
+-------------------+
    |
    v
+-------------------+
| DOM 업데이트       |  render_effect가 DOM 조작
+-------------------+
```

## Version Tracking

시그널은 `rv` (read version)와 `wv` (write version)를 통해 변경을 추적한다. `increment_write_version()` (runtime.js)으로 전역 쓰기 버전이 증가하며, 이를 통해 derived가 재계산이 필요한지 판단한다. 상태 플래그에는 `CLEAN`, `DIRTY`, `MAYBE_DIRTY` 등이 있어 불필요한 재계산을 방지한다.

## Runtime Integration

runtime.js의 핵심 전역 변수:
- `active_reaction` (73행): 현재 실행 중인 reaction (effect 또는 derived)
- `active_effect` (83행): 현재 실행 중인 effect
- `current_sources` (95행): reaction 내에서 생성된 source 목록 (자기 자신의 source 읽기/쓰기 시 재실행 방지)

## Dependencies

- `#client/constants` - 상태 플래그 상수 (CLEAN, DIRTY, DERIVED 등)
- `../shared/utils.js` - 유틸리티 함수
- `../../flags/index.js` - 기능 플래그 (async_mode_flag, tracing_mode_flag)

## Used by

- [[Client-Runtime]] - 컴포넌트 실행 시 반응성 시스템 사용
- [[DOM-Operations]] - render_effect를 통한 DOM 업데이트
- 컴파일러 출력 코드 - `$state`, `$derived`, `$effect` 변환 결과

## 설정

반응성 시스템의 동작을 제어하는 기능 플래그와 설정:

| 플래그 / 설정 | 위치 | 설명 |
|---------------|------|------|
| `SVELTE_NO_ASYNC` | 환경 변수 | `true`로 설정하면 비동기 반응성(`async` 모드)을 비활성화. CI의 TestNoAsync 잡에서 사용 |
| `async_mode_flag` | `src/internal/flags/index.js` | 비동기 파생 및 서스펜스 지원 활성화 플래그 |
| `tracing_mode_flag` | `src/internal/flags/index.js` | `$inspect.trace()` 디버깅 추적 활성화 플래그 |
| `legacy_mode_flag` | `src/internal/flags/index.js` | Svelte 4 호환 모드. `beforeUpdate`/`afterUpdate` 등 레거시 라이프사이클 활성화 |
| `DEV` | `esm-env` | 개발 모드 플래그. 소유권 검증, 추적, 경고 메시지 등 활성화. 프로덕션 빌드에서 트리셰이킹으로 제거 |

> [!note] 플래그 설정 방식
> 기능 플래그는 컴파일러가 `dev` 옵션과 컴포넌트 설정에 따라 생성 코드에 주입한다. `src/internal/flags/index.js`에서 `set_flags()` 함수로 런타임에 플래그를 설정할 수도 있다.

## Testing

```bash
pnpm test signals          # 시그널 단위 테스트
pnpm test runtime-runes    # Runes 런타임 통합 테스트
```

## Related Notes

- [[Client-Runtime]] - 반응성 시스템을 사용하는 클라이언트 런타임
- [[DOM-Operations]] - Effect가 최종적으로 업데이트하는 DOM 레이어
- [[Server-Runtime]] - 서버에서는 반응성 시스템 대신 단순 실행
- [[Practice-Reactivity]] - 연습 문제
