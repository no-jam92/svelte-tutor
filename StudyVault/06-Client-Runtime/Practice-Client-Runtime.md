---
module: client-runtime
type: practice
keywords: runtime, DOM, mount, hydrate, lifecycle, exercises
tags: [module-client, practice]
---

# Practice: Client Runtime & DOM Operations

> [!tip] 학습 방법
> 소스 코드를 직접 열어 함수 호출 흐름을 따라가 보세요. 브라우저 DevTools에서 중단점을 설정하여 실제 실행을 관찰하는 것도 효과적입니다.

## Q1. [trace] mount() 호출 시 내부적으로 어떤 일이 발생하나요?

```javascript
import { mount } from 'svelte';
import App from './App.svelte';

mount(App, { target: document.body });
```

위 코드가 실행될 때의 내부 흐름을 추적하세요.

> [!answer]- 정답 보기
> 1. **`mount()`** (render.js 67행): `_mount(component, options)`를 호출한다.
> 2. **`init_operations()`**: DOM 조작 함수 초기화 (첫 호출 시만). `$window`, `$document` 설정, `firstChild`/`nextSibling` getter 캐싱.
> 3. **이벤트 위임 설정**: `all_registered_events`에 등록된 이벤트에 대해 `target`에 이벤트 리스너 등록 (이벤트 위임 패턴).
> 4. **`component_root()` 호출** (effects.js): `ROOT_EFFECT`를 생성하여 컴포넌트 함수 실행의 루트 effect로 설정.
> 5. **`push(context)`**: 새 `ComponentContext`를 스택에 push.
> 6. **컴포넌트 함수 실행**: `App($$anchor)` 호출. 내부에서 `template()`으로 DOM 생성, `render_effect()`로 동적 바인딩 설정.
> 7. **`pop()`**: 컴포넌트 컨텍스트 스택에서 pop.
> 8. **DOM 삽입**: 생성된 프래그먼트가 `target`에 삽입됨.
> 9. **effect 플러시**: 초기 render_effect들이 실행되어 동적 값이 DOM에 반영됨.

## Q2. [trace] template()이 DOM을 생성하는 과정을 설명하세요

컴파일러가 다음 코드를 어떻게 변환하고, `template()`이 어떻게 DOM을 만드는지 추적하세요.

```svelte
<div class="container">
    <h1>{title}</h1>
    <p>{description}</p>
</div>
```

> [!answer]- 정답 보기
> 1. **컴파일러 출력**:
> ```javascript
> var root = template(`<div class="container"><h1> </h1><p> </p></div>`);
> ```
> 정적인 HTML 구조를 문자열로 표현. 동적인 텍스트 위치에 공백 문자 배치.
>
> 2. **`template()` 호출** (template.js): 문자열을 `innerHTML`로 파싱하여 DOM 노드를 생성하고 캐싱. 이후 호출 시 `cloneNode(true)`로 복제.
>
> 3. **컴포넌트 함수 내부**:
> ```javascript
> var fragment = root();                    // cloneNode로 복제
> var div = get_first_child(fragment);      // <div>
> var h1 = get_first_child(div);           // <h1>
> var text1 = get_first_child(h1);         // " " (텍스트 노드)
> var p = get_next_sibling(h1);            // <p>
> var text2 = get_first_child(p);          // " " (텍스트 노드)
> ```
>
> 4. **render_effect로 동적 바인딩**:
> ```javascript
> render_effect(() => set_text(text1, title));
> render_effect(() => set_text(text2, description));
> ```
> `set_text()` (render.js 46행)는 이전 값과 비교하여 달라진 경우에만 `nodeValue`를 업데이트.
>
> 핵심: `get_first_child()`와 `get_next_sibling()`는 네이티브 DOM getter를 직접 사용하여 성능 최적화 (operations.js).

## Q3. [trace] hydrate()와 mount()의 차이를 코드 수준에서 설명하세요

> [!answer]- 정답 보기
> **mount()의 경우:**
> - `template()`이 `innerHTML`로 새 DOM을 생성.
> - `cloneNode(true)`로 복제 후 `target`에 삽입.
>
> **hydrate()의 경우** (render.js 96행):
> - `set_hydrating(true)`로 하이드레이션 모드 활성화.
> - `template()`이 호출되어도 새 DOM을 생성하지 않고, 기존 SSR HTML에서 노드를 찾아감.
> - `hydrate_node` (hydration.js)가 현재 위치를 추적.
> - `get_next_sibling()`로 다음 노드로 이동하면서 기존 DOM 노드에 반응성/이벤트 연결.
> - `<!--[-->` / `<!--]-->` 하이드레이션 마커로 블록 경계 식별.
> - `recover` 옵션이 `true`이면 하이드레이션 불일치 시 클라이언트 렌더링으로 폴백.
> - `options.intro`가 기본적으로 `false` (하이드레이션 시 인트로 트랜지션 비활성화).

## Q4. [debug] each 블록에서 리스트 업데이트가 느릴 때 어떻게 진단하나요?

`{#each items as item}` 블록에서 1000개 항목의 리스트가 업데이트될 때 성능 문제가 발생한다면?

> [!answer]- 정답 보기
> **진단 체크리스트:**
>
> 1. **key 사용 여부**: `{#each items as item (item.id)}`처럼 고유 key를 지정했는가? key가 없으면 인덱스 기반 비교로 불필요한 업데이트가 발생할 수 있다.
>
> 2. **reconciler 동작 확인**: `reconciler.js`의 키 기반 알고리즘이 최소 DOM 조작을 수행하는지 확인. key가 안정적이지 않으면 (예: 매번 새 객체 생성) 모든 항목이 재생성될 수 있다.
>
> 3. **불필요한 반응성**: each 블록 내부의 `render_effect`가 너무 많은 시그널에 의존하고 있지 않은가? 각 항목의 effect가 리스트 전체 `$state`에 의존하면 하나의 항목 변경이 모든 항목의 effect를 트리거할 수 있다.
>
> 4. **배열 변이 방식**: 전체 배열을 새로 생성 (`[...items, new]`)하면 proxy가 모든 인덱스의 변경을 감지할 수 있다. `items.push(new)`가 proxy 덕분에 더 효율적일 수 있다.
>
> 5. **DevTools 프로파일링**: Chrome Performance 탭에서 `Batch.flush`, `update_effect` 함수의 실행 시간을 측정한다.
>
> 6. **`$inspect` 활용**: `$inspect(items)`로 변경 빈도와 패턴을 확인한다.

## Q5. [extend] 커스텀 블록 타입을 추가하려면 어떤 파일을 수정해야 하나요?

예를 들어 `{#repeat count}...{/repeat}` 같은 새 블록을 추가한다고 가정하세요.

> [!answer]- 정답 보기
> 새 블록 추가는 컴파일러와 런타임 양쪽 수정이 필요하다.
>
> **런타임 (이 모듈):**
> 1. `packages/svelte/src/internal/client/dom/blocks/`에 `repeat.js` 생성.
> 2. 기존 블록 (예: `each.js`)을 참고하여 `BRANCH_EFFECT`를 생성하는 함수 작성.
> 3. `packages/svelte/src/internal/client/index.js`에서 export 추가.
> 4. 서버 측도 필요하면 `packages/svelte/src/internal/server/blocks/`에도 추가.
>
> **컴파일러:**
> 1. 파서: `packages/svelte/src/compiler/phases/1-parse/`에서 `{#repeat}` 문법 인식.
> 2. 분석: `packages/svelte/src/compiler/phases/2-analyze/`에서 의미 분석.
> 3. 코드 생성 (클라이언트): `packages/svelte/src/compiler/phases/3-transform/client/`에서 런타임 호출 코드 생성.
> 4. 코드 생성 (서버): `packages/svelte/src/compiler/phases/3-transform/server/`에서 SSR 코드 생성.
>
> **기존 블록 참고**: `blocks/` 디렉토리의 파일들 (`if.js`, `each.js`, `key.js` 등)이 `BRANCH_EFFECT`를 어떻게 생성하고 관리하는지 패턴을 따른다.

## Q6. [trace] set_text()의 최적화 메커니즘을 설명하세요

> [!answer]- 정답 보기
> `set_text()` (render.js 46-55행):
>
> ```javascript
> export function set_text(text, value) {
>     var str = value == null ? '' : typeof value === 'object' ? `${value}` : value;
>     if (str !== (text.__t ??= text.nodeValue)) {
>         text.__t = str;
>         text.nodeValue = `${str}`;
>     }
> }
> ```
>
> 최적화 포인트:
> 1. **캐싱 (`__t`)**: 텍스트 노드에 `__t` 프로퍼티로 현재 값을 캐싱. `nodeValue`를 매번 읽는 것보다 빠름.
> 2. **비교 후 업데이트**: 값이 같으면 DOM 접근 자체를 하지 않음. DOM 쓰기는 비용이 크므로 이 비교가 중요함.
> 3. **nullish 처리**: `null`/`undefined`를 빈 문자열로 변환.
> 4. **객체 문자열화**: 객체는 `${value}`로 명시적 변환 (proxy된 배열 등의 반응성 보존).
>
> `__t`는 `init_operations()` (operations.js 63-66행)에서 Text.prototype에 `undefined`로 미리 정의하여 hidden class 최적화를 돕는다.

## Related Notes

- [[Client-Runtime]] - 런타임 전체 아키텍처
- [[DOM-Operations]] - DOM 조작 상세
- [[Reactivity-System]] - DOM 업데이트를 트리거하는 반응성
- [[Practice-Reactivity]] - 반응성 연습 문제
