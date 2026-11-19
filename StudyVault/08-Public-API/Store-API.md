---
module: store
path: packages/svelte/src/store
keywords: store, writable, readable, derived, subscription
tags: [module-store, api-store]
---

# Store API

> [!info] 학습 목표
> Svelte Store --- 컴포넌트 간 상태 공유를 위한 반응성 저장소 시스템을 이해한다.

**난이도:** ★★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `src/store/index-client.js` | 클라이언트 측 store 구현 (toStore, fromStore) |
| `src/store/index-server.js` | 서버 측 store 구현 |
| `src/store/shared/index.js` | 공유 구현 (writable, readable, derived, get) |
| `src/store/public.d.ts` | Readable, Writable 타입 정의 |

## Store 아키텍처

```
+--------------------------------------------------+
|               svelte/store                        |
+--------------------------------------------------+
|                                                   |
|  Core Stores (shared/index.js)                    |
|  +---------------------------------------------+ |
|  | writable(value, start?)  --> Writable<T>     | |
|  | readable(value, start?)  --> Readable<T>     | |
|  | derived(stores, fn)      --> Readable<T>     | |
|  | readonly(store)          --> Readable<T>     | |
|  | get(store)               --> T (즉시 값)     | |
|  +---------------------------------------------+ |
|                                                   |
|  Runes <-> Store Bridge (index-client.js)         |
|  +---------------------------------------------+ |
|  | toStore(get, set?)  Runes --> Store           | |
|  | fromStore(store)    Store --> { current: V }  | |
|  +---------------------------------------------+ |
+--------------------------------------------------+
```

## Core Store 함수

### writable(value, start?)

읽기/쓰기 가능한 store를 생성한다.

```js
import { writable } from 'svelte/store';

const count = writable(0);
count.set(1);          // 값 설정
count.update(n => n + 1); // 함수로 업데이트
```

### readable(value, start?)

읽기 전용 store. 외부에서 값을 변경할 수 없고, `start` 콜백 내부에서만 값을 설정한다.

### derived(stores, fn)

하나 이상의 store에서 파생된 값을 계산한다.

```js
import { derived } from 'svelte/store';

const doubled = derived(count, $count => $count * 2);
```

### get(store)

store의 현재 값을 동기적으로 가져온다. 내부적으로 subscribe/unsubscribe를 즉시 수행한다.

## Runes <-> Store Bridge

### toStore(get, set?)

Runes 기반 반응성 상태를 Store 인터페이스로 변환한다.

```js
import { toStore } from 'svelte/store';

let count = $state(0);
const store = toStore(
  () => count,        // getter
  (v) => (count = v)  // setter (optional)
);
```

> [!note] 내부 구현
> `toStore`는 `effect_root` + `render_effect`를 사용하여 Runes의 반응성을 Store의 subscription 모델로 브릿지한다. getter 값이 변경되면 `render_effect`가 트리거되어 Store의 `set`을 호출한다.

### fromStore(store)

Store를 `{ current: V }` 형태의 반응성 객체로 변환한다. Writable store인 경우 `current`에 값을 쓸 수도 있다.

```js
import { fromStore, writable } from 'svelte/store';

const store = writable(0);
const count = fromStore(store);

count.current; // 0
store.set(1);
count.current; // 1

count.current += 1; // Writable인 경우 가능
```

> [!note] 내부 구현
> `fromStore`는 `createSubscriber`를 사용하여 effect tracking 환경에서 자동 구독/해제를 관리한다. effect tracking 외부에서는 `get(store)`로 폴백한다.

## Subscription 모델

### $store 구문

`.svelte` 파일에서 `$` 접두사로 store를 자동 구독할 수 있다. 이것은 **컴파일러 매직**이다.

```svelte
<script>
  import { writable } from 'svelte/store';
  const count = writable(0);
  // $count는 자동 구독된 값
</script>
<p>{$count}</p>
```

컴파일러는 이를 다음과 같이 변환한다:
1. 컴포넌트 초기화 시 `store.subscribe()` 호출
2. 로컬 변수에 값 할당
3. 컴포넌트 소멸 시 unsubscribe

### Store Contract

Svelte store는 간단한 인터페이스를 따른다:

```ts
interface Readable<T> {
  subscribe(run: (value: T) => void): () => void;
}

interface Writable<T> extends Readable<T> {
  set(value: T): void;
  update(updater: (value: T) => T): void;
}
```

> [!tip] 커스텀 Store
> `subscribe` 메서드만 구현하면 어떤 객체든 Store로 사용할 수 있다. RxJS Observable 등을 래핑하는 것도 가능하다.

## Client vs Server 구현

| 기능 | Client | Server |
|------|--------|--------|
| `writable/readable/derived` | shared 구현 공유 | shared 구현 공유 |
| `toStore` | effect_root 기반 반응성 브릿지 | 단순 동기 구현 |
| `fromStore` | createSubscriber 기반 | 단순 동기 구현 |
| `get` | 즉시 subscribe/unsubscribe | 동일 |

## 테스팅
- 실행: `pnpm test store`
- 패턴: 단위 테스트 (Vitest)
- 테스트 위치: `packages/svelte/tests/store/`

## Related Notes

- [[Public-API]] --- 전체 공개 API 개요
- [[Motion-API]] --- Spring/Tweened (내부적으로 writable 사용)
- [[Reactivity-System]] --- Runes 반응성 시스템
- [[Practice-Public-API]] --- 연습문제
