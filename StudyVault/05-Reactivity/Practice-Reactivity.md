---
module: reactivity
type: practice
keywords: signals, effects, derived, debugging, exercises
tags: [module-reactivity, practice]
---

# Practice: Reactivity System

> [!tip] 학습 방법
> 각 문제를 먼저 스스로 생각해 본 후, 접힌 답변을 펼쳐보세요. 소스 코드를 직접 따라가며 확인하는 것이 가장 효과적입니다.

## Q1. [trace] $state로 생성된 시그널이 변경될 때 내부적으로 어떤 일이 발생하나요?

`count`를 `$state(0)`으로 선언하고 `count = 5`로 변경할 때의 전체 흐름을 추적하세요.

> [!answer]- 정답 보기
> 1. **컴파일러 변환**: `count = 5`는 `set(count_signal, 5)`로 변환된다.
> 2. **`set()` 호출** (sources.js): 내부적으로 `internal_set()`을 호출한다.
> 3. **값 비교**: `signal.equals(signal.v, value)`로 이전 값과 비교. 같으면 조기 반환.
> 4. **값 업데이트**: `signal.v = value`로 새 값 저장.
> 5. **write version 증가**: `increment_write_version()`으로 전역 쓰기 버전 증가, `signal.wv` 업데이트.
> 6. **reactions 순회**: `signal.reactions` 목록을 순회하며 각 reaction을 DIRTY로 표시.
> 7. **effect 스케줄링**: DIRTY로 표시된 effect들을 `schedule_effect()`를 통해 `Batch`에 등록.
> 8. **배치 플러시**: microtask에서 `Batch.flush()`가 실행되어 등록된 모든 effect를 재실행.
> 9. **DOM 업데이트**: render_effect가 재실행되면서 DOM에 새 값 반영.
>
> 핵심 파일 경로: `packages/svelte/src/internal/client/reactivity/sources.js`

## Q2. [trace] $derived 값이 재계산되는 과정을 추적하세요

```svelte
let count = $state(0);
let doubled = $derived(count * 2);
```

`doubled`이 템플릿에서 읽힐 때 어떤 일이 발생하는지 설명하세요.

> [!answer]- 정답 보기
> 1. **lazy 평가**: `derived()` 호출 시점에는 계산하지 않는다. `f: DERIVED | DIRTY`, `v: UNINITIALIZED`로 초기화.
> 2. **읽기 시점**: 템플릿의 render_effect가 `get(doubled_signal)`을 호출하면, DIRTY 플래그가 감지된다.
> 3. **`execute_derived()` 호출** (deriveds.js): `signal.fn()`을 실행하여 계산.
> 4. **의존성 수집**: 계산 함수 내에서 `get(count_signal)`이 호출되면, `count_signal.reactions`에 `doubled_signal`이 등록된다.
> 5. **값 캐싱**: 계산 결과를 `signal.v`에 저장하고 상태를 CLEAN으로 변경.
> 6. **이후 읽기**: CLEAN 상태이면 `signal.v`를 즉시 반환 (재계산 없음).
> 7. **무효화**: `count`가 변경되면 `doubled_signal`이 MAYBE_DIRTY로 표시. 다음 읽기 시 의존성 버전을 확인하여 실제로 DIRTY인지 판단.
>
> 핵심 파일 경로: `packages/svelte/src/internal/client/reactivity/deriveds.js`

## Q3. [trace] 배칭(batching)이 어떻게 여러 상태 변경을 하나의 업데이트로 묶나요?

```svelte
function handleClick() {
    count = 1;
    name = "hello";
    items = [...items, newItem];
}
```

위 핸들러에서 3개의 상태가 변경될 때 DOM 업데이트는 몇 번 발생하나요?

> [!answer]- 정답 보기
> **1번만 발생한다.** 그 과정은 다음과 같다:
>
> 1. `count = 1` 실행: `set()` -> `internal_set()` -> effect들을 DIRTY로 표시 -> `schedule_effect()`로 Batch에 등록. 이때 microtask가 예약되지만 아직 실행되지 않음.
> 2. `name = "hello"` 실행: 동일한 과정. 같은 Batch에 추가 effect 등록.
> 3. `items = [...]` 실행: 동일한 과정. 같은 Batch에 추가 effect 등록.
> 4. 이벤트 핸들러 함수가 완전히 끝난 후, microtask 큐에서 `Batch.flush()` 실행.
> 5. Batch에 등록된 모든 effect를 한 번에 처리. 같은 effect가 여러 번 등록되어도 한 번만 실행.
>
> `flushSync()`를 호출하면 즉시 동기적으로 플러시할 수 있다.
>
> 핵심 파일 경로: `packages/svelte/src/internal/client/reactivity/batch.js`

## Q4. [debug] 반응성이 예상대로 작동하지 않을 때 어디를 확인해야 하나요?

컴포넌트에서 `$state` 값을 변경했는데 UI가 업데이트되지 않는 경우의 디버깅 체크리스트를 작성하세요.

> [!answer]- 정답 보기
> **디버깅 체크리스트:**
>
> 1. **Proxy 대상 확인**: `$state`에 할당된 객체가 Proxy 가능한 타입인가? (순수 객체/배열만 가능, `Map`, `Set`, 클래스 인스턴스는 불가). `proxy.js` 42-49행 참고.
>
> 2. **참조 vs 변이**: 객체의 프로퍼티를 직접 변경(mutation)하고 있는가, 아니면 새 객체를 할당하는가? Proxy 덕분에 직접 변이도 추적되지만, `$state`가 아닌 일반 변수의 프로퍼티 변이는 추적되지 않는다.
>
> 3. **equals 함수**: `signal.equals` (기본: `Object.is`)가 이전/이후 값을 같다고 판단하고 있지 않은가? `equality.js` 참고.
>
> 4. **untrack 컨텍스트**: `untrack()` 내부에서 시그널을 읽고 있지 않은가? 의존성이 등록되지 않는다.
>
> 5. **Effect 외부**: `$effect`나 render context 외부에서 시그널을 읽으면 반응성이 작동하지 않는다. `validate_effect()` (effects.js 53행)가 이를 감지한다.
>
> 6. **비동기 컨텍스트**: `await` 이후에 시그널을 읽으면 reactivity loss가 발생할 수 있다. `reactivity_loss_tracker` (deriveds.js)가 dev 모드에서 경고한다.
>
> 7. **DEV 도구 활용**: `$inspect()` rune을 사용하여 시그널 값 변화를 추적한다.

## Q5. [extend] 새로운 반응성 컬렉션(예: ReactiveWeakMap)을 추가하려면?

Svelte 5의 반응성 시스템을 확장하여 `WeakMap`을 반응적으로 만드는 방법을 설계하세요.

> [!answer]- 정답 보기
> **설계 접근법:**
>
> Proxy는 `WeakMap`에 사용할 수 없다 (proxy.js 48행에서 `object_prototype`/`array_prototype`만 허용). 따라서 래퍼 클래스를 만들어야 한다.
>
> ```javascript
> import { source, set } from './reactivity/sources.js';
> import { get } from './runtime.js';
>
> class ReactiveWeakMap {
>     #map = new WeakMap();
>     #sources = new WeakMap(); // key별 시그널
>     #version = source(0);    // 전체 변경 추적
>
>     get(key) {
>         // 개별 key의 source를 읽어 의존성 등록
>         let s = this.#sources.get(key);
>         if (s) get(s);
>         return this.#map.get(key);
>     }
>
>     set(key, value) {
>         this.#map.set(key, value);
>         let s = this.#sources.get(key);
>         if (!s) {
>             s = source(0);
>             this.#sources.set(key, s);
>         }
>         set(s, get(s) + 1); // 변경 알림
>         return this;
>     }
>
>     has(key) {
>         let s = this.#sources.get(key);
>         if (s) get(s);
>         return this.#map.has(key);
>     }
>
>     delete(key) {
>         let s = this.#sources.get(key);
>         if (s) set(s, get(s) + 1);
>         return this.#map.delete(key);
>     }
> }
> ```
>
> **핵심 원리**: 각 key마다 `source()` 시그널을 생성하고, 읽기 시 `get()`으로 의존성을 등록하고, 쓰기 시 `set()`으로 변경을 알린다. 이 패턴은 `proxy.js`에서 객체 프로퍼티마다 `sources` Map을 유지하는 것과 동일하다.

## Q6. [trace] Proxy 객체에서 새 프로퍼티를 추가할 때 내부 동작을 추적하세요

```svelte
let obj = $state({ a: 1 });
obj.b = 2;  // 새 프로퍼티 추가
```

> [!answer]- 정답 보기
> 1. `obj`는 `proxy()`로 감싸져 있으며, 내부에 `sources` Map과 `version` source를 가진다.
> 2. `obj.b = 2` 실행 시 Proxy의 `set` 트랩이 호출된다.
> 3. `sources.get('b')`를 조회하면 `undefined` (새 프로퍼티이므로).
> 4. `with_parent()` 컨텍스트에서 `source(2)`를 호출하여 새 시그널 생성 후 `sources.set('b', signal)`로 저장.
> 5. `version` source를 `increment()`하여 객체 전체를 관찰하는 effect들에게 변경 알림.
> 6. 이전에 `obj.b`를 읽은 적이 있는 effect가 있다면(그 시점에는 undefined였을 것), 새 source의 변경으로 인해 재실행된다.
>
> `with_parent()` (proxy.js 65-84행)는 자식 source가 현재 reaction이 아닌 원래 proxy가 생성된 reaction 컨텍스트에서 만들어지도록 보장한다.

## Related Notes

- [[Reactivity-System]] - 이 문제들의 배경 지식
- [[Client-Runtime]] - Effect가 실행되는 런타임 환경
- [[Practice-Client-Runtime]] - 클라이언트 런타임 연습 문제
