---
module: practice-public-api
path: 08-Public-API
keywords: practice, API, lifecycle, store, motion, runes
tags: [api-runtime, practice]
---

# Practice: Public API

> [!info] 연습 가이드
> 아래 문제들을 통해 Svelte 5의 공개 API에 대한 이해를 점검하세요. 접힌 답변을 열기 전에 먼저 직접 생각해 보세요.

## Q1. onMount vs $effect --- 차이점은?

`onMount`와 `$effect`는 모두 컴포넌트 마운트 시 코드를 실행할 수 있다. 두 가지의 핵심적인 차이점은 무엇이며, 언제 어떤 것을 선택해야 하는가?

> [!answer]- 정답 보기
> **실행 횟수:** `onMount`는 **한 번만** 실행되고, `$effect`는 의존성이 변경될 때마다 **재실행**된다.
>
> **SSR 동작:** `onMount`는 서버에서 실행되지 않지만 (`index-server.js`에서 noop), `$effect`도 서버에서 실행되지 않는다.
>
> **내부 구현:** `onMount(fn)`은 내부적으로 `user_effect(() => { const cleanup = untrack(fn); ... })`로 구현되어 있다. `untrack`으로 감싸기 때문에 fn 내부의 반응성 읽기가 의존성으로 추적되지 않아 재실행이 없다.
>
> **선택 기준:**
> - 초기 설정, DOM 조작, 외부 라이브러리 초기화 --> `onMount`
> - 반응성 데이터에 따른 부수 효과 --> `$effect`

## Q2. toStore와 fromStore는 왜 필요한가?

Svelte 5에서 Runes (`$state`)와 Store (`writable`)를 브릿지하는 `toStore`와 `fromStore`의 존재 이유는 무엇인가? 각각의 내부 동작을 설명하시오.

> [!answer]- 정답 보기
> **존재 이유:** Svelte 5는 Runes 기반 반응성을 도입했지만, 기존 라이브러리와 코드는 Store 인터페이스를 사용한다. 두 시스템 간의 **상호운용성**이 필요하다.
>
> **toStore(get, set?):** Runes --> Store 방향. `effect_root` + `render_effect`를 사용하여 getter 값의 변경을 감지하고, 변경 시 writable store의 `set`을 호출한다. subscriber가 있을 때만 effect가 활성화된다.
>
> **fromStore(store):** Store --> Runes 방향. `createSubscriber`를 사용하여 `{ current: V }` 형태의 반응성 객체를 반환한다. effect tracking 환경에서는 자동 구독되고, 그 외에는 `get(store)`로 동기적으로 값을 가져온다.

## Q3. Spring과 Tweened의 애니메이션 모델 차이

Spring과 Tweened는 모두 값을 부드럽게 변화시키지만 근본적으로 다른 애니메이션 모델을 사용한다. 각각의 모델을 설명하고, 어떤 상황에서 어떤 것이 적합한지 설명하시오.

> [!answer]- 정답 보기
> **Spring --- 물리 기반:**
> - 감쇠 스프링 시뮬레이션 (`acceleration = stiffness * delta - damping * velocity`)
> - **지속 시간이 정해져 있지 않다** --- 물리적으로 정지할 때까지 계속
> - stiffness, damping, precision 파라미터로 제어
> - 사용자 인터랙션 추적 (드래그, 마우스 따라가기)에 자연스러움
>
> **Tweened --- 시간 기반:**
> - 정해진 duration 동안 이징 함수로 보간
> - `get_interpolator`가 타입별 보간 전략 선택 (숫자, Date, 배열, 객체)
> - duration, delay, easing, interpolate 파라미터로 제어
> - 명확한 시작/끝이 있는 전환 (페이지 전환, 상태 변경)에 적합
>
> **공통점:** 둘 다 `writable` store 기반, `raf` 루프 사용, 재귀적 타입 처리

## Q4. flushSync()는 언제, 왜 사용하는가?

`flushSync()`의 역할과 사용 시나리오를 설명하시오. 사용하지 않으면 어떤 일이 발생하는가?

> [!answer]- 정답 보기
> **역할:** Svelte의 반응성 업데이트는 기본적으로 **배치(batch) 처리**된다. 여러 상태 변경이 하나의 마이크로태스크에서 모아져 한 번에 DOM에 반영된다. `flushSync()`는 이 대기열을 **즉시 동기적으로** 처리한다.
>
> **사용 시나리오:**
> - 상태 변경 직후 DOM 측정이 필요할 때 (scrollHeight, getBoundingClientRect 등)
> - 테스트에서 상태 변경 후 DOM 상태를 검증할 때
> - 애니메이션 프레임 사이에 정확한 타이밍이 필요할 때
>
> **사용하지 않으면:** 상태 변경과 DOM 업데이트 사이에 시간 차이가 있어, 변경 직후 DOM을 읽으면 이전 상태의 값을 얻게 된다.

## Q5. getAbortSignal()의 동작 원리

`getAbortSignal()` 함수는 어떻게 동작하며, 왜 유용한가? `index-client.js`의 구현을 참고하여 설명하시오.

> [!answer]- 정답 보기
> **동작 원리:**
> ```js
> export function getAbortSignal() {
>   if (active_reaction === null) {
>     e.get_abort_signal_outside_reaction();
>   }
>   return (active_reaction.ac ??= new AbortController()).signal;
> }
> ```
> - 현재 활성 reaction (derived 또는 effect)의 `AbortController`에서 signal을 반환
> - reaction이 재실행되거나 소멸되면 AbortController가 `abort()`되어 signal이 취소됨
> - `??=`로 lazy 초기화하여 필요할 때만 AbortController 생성
>
> **유용성:** `$derived` 내에서 fetch 호출 시, 의존성이 변경되어 이전 fetch가 불필요해지면 자동으로 취소된다. race condition을 방지하는 우아한 패턴이다.

## Q6. Rune이 .svelte 파일 외부에서 사용되면?

DEV 모드에서 Rune ($state, $effect 등)이 `.svelte` 파일 외부의 일반 JS에서 사용되면 어떤 일이 발생하는가? 그 메커니즘을 설명하시오.

> [!answer]- 정답 보기
> `index-client.js` 상단의 `throw_rune_error` 함수가 DEV 모드에서 `globalThis`에 각 rune 이름을 `Object.defineProperty`로 등록한다.
>
> ```js
> Object.defineProperty(globalThis, rune, {
>   configurable: true,
>   get: () => {
>     if (value !== undefined) return value;
>     e.rune_outside_svelte(rune); // 에러 발생
>   },
>   set: (v) => { value = v; }
> });
> ```
>
> `.svelte` 파일에서는 컴파일러가 rune을 실제 런타임 코드로 변환하므로 globalThis에 접근하지 않는다. 하지만 일반 `.js` 파일에서 `$state`를 쓰면 globalThis의 getter가 호출되어 `rune_outside_svelte` 에러가 발생한다.
>
> 이를 통해 개발자에게 "이 구문은 Svelte 파일 내에서만 사용할 수 있다"는 명확한 에러 메시지를 제공한다.

## Q7. Store의 subscribe 계약

Svelte Store가 되려면 어떤 인터페이스를 구현해야 하는가? 외부 라이브러리(예: RxJS Observable)를 Svelte Store로 사용하려면 어떻게 해야 하는가?

> [!answer]- 정답 보기
> **Store Contract:**
> `subscribe(run: (value: T) => void): () => void` 메서드만 있으면 된다.
> - `run`은 값이 변경될 때 호출되는 콜백
> - 반환값은 구독 해제 함수 (unsubscribe)
>
> **RxJS 래핑 예시:**
> ```js
> function fromObservable(observable) {
>   return {
>     subscribe(run) {
>       const sub = observable.subscribe(run);
>       return () => sub.unsubscribe();
>     }
>   };
> }
> ```
>
> 이렇게 하면 `.svelte` 파일에서 `$store` 구문으로 자동 구독이 가능하다. Writable로 만들려면 `set`과 `update` 메서드도 추가해야 한다.

## Related Notes

- [[Public-API]] --- 공개 API 전체 개요
- [[Store-API]] --- Store 시스템 상세
- [[Motion-API]] --- 애니메이션 모듈 상세
- [[Runtime-Exercises]] --- 런타임 연습문제
- [[Reactivity-Exercises]] --- 반응성 연습문제
