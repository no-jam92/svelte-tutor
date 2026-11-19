---
module: exercises
path: 10-Exercises
keywords: practice, onboarding, reactivity, signals, state, derived, effect
tags: [module-reactivity, practice]
---

# 반응성 --- 온보딩 연습문제

> [!info] 연습 가이드
> 아래 문제들을 통해 Svelte 5의 반응성 시스템(Signals)에 대한 이해를 점검하세요. `[trace]`는 코드 추적, `[debug]`는 디버깅, `[extend]`는 확장 과제입니다.

## 연습 1 [trace]: $state(0)의 내부 데이터 구조

`$state(0)`이 호출될 때 내부적으로 생성되는 데이터 구조를 설명하세요.

> [!answer]- 정답 보기
> `$state(0)`은 컴파일러에 의해 `$.source(0)` (또는 `$.state(0)`)으로 변환된다.
>
> **생성되는 Signal 객체:**
> ```
> Source {
>   v: 0,              // 현재 값 (value)
>   reactions: null,   // 이 소스를 구독하는 reaction(effect/derived) 목록
>   equals: Object.is, // 값 비교 함수
>   version: 0         // 변경 카운터 (dirty 체크용)
> }
> ```
>
> **핵심 파일:** `src/internal/client/reactivity/sources.js`
>
> **동작 방식:**
> - `$.get(source)`: 현재 `active_reaction`이 있으면 의존성으로 등록, 값 반환
> - `$.set(source, value)`: `equals`로 비교 후 다르면 값 업데이트, `version` 증가, 의존 reaction들에 알림
> - 객체/배열의 경우 `$.state()`는 Proxy를 사용하여 깊은 반응성 제공

## 연습 2 [trace]: $derived의 의존성 추적

`$derived(() => a + b)`에서 의존성이 추적되는 과정을 단계별로 추적하세요.

> [!answer]- 정답 보기
> `$derived(() => a + b)`는 컴파일러에 의해 `$.derived(() => $.get(a) + $.get(b))`로 변환된다.
>
> **1단계: Derived 생성**
> ```
> Derived {
>   fn: () => $.get(a) + $.get(b),  // 계산 함수
>   v: undefined,                     // 캐시된 값 (lazy)
>   deps: null,                       // 의존성 목록
>   version: 0,                       // dirty 체크용
>   dirty: true                       // 초기 상태는 dirty
> }
> ```
>
> **2단계: 최초 평가 (lazy)**
> - 누군가 derived 값을 읽을 때 (`$.get(derived)`) 처음 평가
> - `active_reaction`을 현재 derived로 설정
> - fn 실행: `$.get(a)` --> a의 reactions에 이 derived 등록
> - fn 실행: `$.get(b)` --> b의 reactions에 이 derived 등록
> - deps = [a, b]로 설정, 결과값 캐시
>
> **3단계: 재평가 트리거**
> - `a` 또는 `b`가 변경되면 derived를 dirty로 표시
> - 다음 읽기 시 fn을 재실행하여 새 값 계산
> - 의존성이 바뀌면 (조건부 읽기 등) deps 목록 갱신
>
> **핵심 파일:** `src/internal/client/reactivity/deriveds.js`

## 연습 3 [trace]: flushSync()의 배치 처리

`flushSync()`가 호출될 때 배치 처리 과정을 추적하세요.

```js
let count = $state(0);

count = 1;
count = 2;
count = 3;
flushSync(); // 이 시점에서 무슨 일이?
```

> [!answer]- 정답 보기
> **배치(Batch) 처리 과정:**
>
> **1. 상태 변경 큐잉** (`count = 1, 2, 3`)
> - 각 `$.set()` 호출마다:
>   - source의 값을 즉시 업데이트 (v = 1, 2, 3)
>   - source의 version 증가
>   - 의존 reaction들을 **dirty로 표시** (아직 실행하지 않음)
>   - 스케줄러에 flush 예약 (마이크로태스크)
>
> **2. flushSync() 호출**
> - 대기 중인 마이크로태스크를 기다리지 않고 **즉시** flush 실행
>
> **3. Flush 과정** (`src/internal/client/runtime.js`)
> ```
> while (dirty_effects.length > 0) {
>   effect = dirty_effects.shift();
>   // dirty 상태인 deps 확인
>   // 실제로 값이 바뀌었는지 재확인
>   // effect의 콜백 실행
> }
> ```
>
> **4. DOM 업데이트**
> - render_effect들이 실행되어 DOM에 최종값(3)만 반영
> - 중간값(1, 2)에 대한 DOM 업데이트는 발생하지 않음
>
> **핵심:** count가 세 번 변경되었지만, DOM 업데이트는 **한 번만** 발생한다. flushSync()는 이 배치를 즉시 강제 실행한다.

## 연습 4 [debug]: $effect 무한 루프 디버깅

`$effect`가 무한 루프에 빠지면 어떻게 디버깅하는지 설명하세요.

```js
let count = $state(0);
$effect(() => {
  count = count + 1; // 위험!
});
```

> [!answer]- 정답 보기
> **문제 원인:**
> - `$effect` 내에서 `count`를 **읽고** (의존성 등록) 동시에 **쓰는** (상태 변경) 패턴
> - 읽기 --> 의존성 등록 --> 쓰기 --> 변경 감지 --> effect 재실행 --> 무한 반복
>
> **Svelte의 보호 메커니즘:**
> - 런타임에 effect 재실행 횟수를 추적
> - 임계값 초과 시 `"Effect recursion limit exceeded"` 에러 발생
> - `src/internal/client/reactivity/effects.js`에서 관리
>
> **디버깅 방법:**
>
> 1. **$inspect 활용:**
>    ```js
>    $inspect(count); // 값 변경을 콘솔에 추적
>    ```
>
> 2. **untrack 사용:**
>    ```js
>    $effect(() => {
>      count = untrack(() => count) + 1; // 읽기를 추적하지 않음
>    });
>    ```
>
> 3. **패턴 수정:**
>    - effect 내에서 읽는 상태를 같은 effect에서 쓰지 않기
>    - `$derived`로 대체 가능한지 검토
>    - 필요하면 `untrack()`으로 읽기 의존성 제거
>
> 4. **소스 코드 확인:**
>    - `src/internal/client/runtime.js` --- 스케줄러의 무한 루프 감지 로직
>    - `src/internal/client/errors.js` --- 에러 메시지 정의

## 연습 5 [extend]: 새로운 반응성 프리미티브

새로운 반응성 프리미티브(예: `$history` --- 값의 변경 이력을 추적하는 상태)를 만들려면 어떤 파일들을 수정해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **방법 A: 런타임 레벨 (유저랜드 라이브러리)**
>
> Runes를 조합하여 구현 (파일 수정 불필요):
> ```js
> // history.svelte.js
> export function createHistory(initial) {
>   let current = $state(initial);
>   let history = $state([initial]);
>
>   return {
>     get value() { return current; },
>     set value(v) { current = v; history.push(v); },
>     get history() { return history; },
>     undo() { if (history.length > 1) { history.pop(); current = history.at(-1); } }
>   };
> }
> ```
>
> **방법 B: 컴파일러 레벨 (새 Rune 추가)**
>
> 수정해야 할 파일:
>
> 1. **파서** (`src/compiler/phases/1-parse/`)
>    - `$history`를 rune으로 인식하도록 등록
>
> 2. **분석기** (`src/compiler/phases/2-analyze/`)
>    - `$history` 사용의 유효성 검사
>    - 반응성 바인딩 타입 분류
>
> 3. **변환기** (`src/compiler/phases/3-transform/`)
>    - `$history(value)`를 런타임 호출로 변환하는 코드 생성
>
> 4. **런타임** (`src/internal/client/reactivity/`)
>    - `history.js` --- 이력 추적 로직 구현
>    - source 기반으로 값 변경 시마다 이력 배열에 기록
>
> 5. **타입** (`src/index.d.ts` 등)
>    - `$history` rune의 TypeScript 타입 정의
>
> 6. **DEV 보호** (`src/index-client.js`)
>    - `throw_rune_error('$history')` 추가
>
> 7. **테스트** (`tests/runtime-runes/samples/`, `tests/signals/`)
>
> > [!tip] 실무 권장
> > 대부분의 경우 방법 A(유저랜드)로 충분하다. 컴파일러 수정은 코어 팀의 RFC 프로세스를 거쳐야 한다.

## 연습 6 [trace]: Proxy 기반 깊은 반응성

`$state({ nested: { value: 0 } })`에서 `obj.nested.value = 1`이 반응성을 트리거하는 과정을 추적하세요.

> [!answer]- 정답 보기
> **1. 초기화:**
> - `$state(obj)`는 `$.source(obj)`를 호출하되, 객체인 경우 **Proxy**로 래핑
> - 중첩 객체 `nested`도 접근 시 재귀적으로 Proxy 래핑
>
> **2. Proxy 동작:**
> ```
> Proxy Handler {
>   get(target, key) {
>     // 'nested' 접근 시 해당 속성도 Proxy로 래핑하여 반환
>     // 현재 active_reaction이 있으면 의존성 등록
>     track(target, key);
>     return wrap_proxy(target[key]);
>   },
>   set(target, key, value) {
>     // 값이 다르면 업데이트
>     target[key] = value;
>     // 의존 reaction들에 알림
>     trigger(target, key);
>     return true;
>   }
> }
> ```
>
> **3. `obj.nested.value = 1` 추적:**
> - `obj.nested` --> Proxy get 트랩 --> nested Proxy 반환
> - `.value = 1` --> nested Proxy의 set 트랩 --> trigger 호출
> - 의존하는 effect/derived들이 dirty로 표시
> - 다음 flush에서 DOM 업데이트
>
> **핵심 파일:** `src/internal/client/proxy.js`

> [!summary]- 학습 포인트 요약
> - **Source**: 반응성의 기본 단위. 값 + version + reactions 목록으로 구성
> - **Derived**: lazy 평가. 의존성이 변경되면 dirty 표시, 다음 읽기 시 재계산
> - **Effect**: eager 실행. 의존성 변경 시 스케줄러 큐에 등록되어 flush 시 실행
> - **배치 처리**: 여러 상태 변경을 모아 한 번에 DOM 업데이트. flushSync()로 강제 즉시 실행
> - **깊은 반응성**: Proxy를 사용하여 중첩 객체의 속성 변경도 자동 추적
> - **무한 루프 보호**: 런타임에서 재귀 실행 횟수를 감시하여 무한 루프 방지

## Related Modules

- [[Reactivity-System]] --- 반응성 시스템 상세
- [[Public-API]] --- flushSync, untrack 등 공개 API
- [[Compiler-Exercises]] --- 컴파일러 연습문제
- [[Runtime-Exercises]] --- 런타임 연습문제
- [[Integration-Exercises]] --- 통합 연습문제
