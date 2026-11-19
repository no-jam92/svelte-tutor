---
module: client-dom
path: packages/svelte/src/internal/client/dom
keywords: DOM, template, blocks, elements, reconciler, hydration
tags: [module-client, arch-runtime]
---

# DOM Operations

> **Purpose**: DOM 조작, 템플릿 처리, 블록 구현을 담당하는 모듈. 컴파일된 Svelte 컴포넌트가 실제로 DOM을 생성하고 업데이트하는 모든 로직이 여기에 있다.

> [!important] 중요도: ★★★
> Svelte가 Virtual DOM 없이 효율적으로 DOM을 업데이트하는 방법의 핵심이다.

## Key Files

| 파일 | 역할 | 핵심 export |
|------|------|-------------|
| `operations.js` | 기본 DOM 조작 (노드 탐색, 생성) | `init_operations()`, `create_text()`, `get_first_child()`, `get_next_sibling()` |
| `template.js` | 템플릿 기반 DOM 프래그먼트 생성 | `template()`, `assign_nodes()` |
| `reconciler.js` | 리스트(each) 효율적 DOM 업데이트 | 키 기반 reconciliation |
| `hydration.js` | SSR HTML 하이드레이션 | `hydrate_node`, `hydrating`, `set_hydrate_node()` |
| `css.js` | 스코프드 CSS 처리 | CSS 클래스 주입 |
| `task.js` | 비동기 태스크 스케줄링 | microtask 기반 업데이트 |

## Blocks

블록은 Svelte의 제어 흐름 구조에 대응하는 DOM 렌더링 단위이다.

| 파일 | 대응 구문 | 설명 |
|------|-----------|------|
| `if.js` | `{#if}` / `{:else}` | 조건부 렌더링 |
| `each.js` | `{#each}` | 리스트 반복 렌더링 |
| `await.js` | `{#await}` | Promise 상태별 렌더링 |
| `key.js` | `{#key}` | 키 변경 시 블록 재생성 |
| `snippet.js` | `{#snippet}` / `{@render}` | 재사용 가능한 마크업 조각 |
| `slot.js` | `<slot>` | 레거시 슬롯 (Svelte 4 호환) |
| `boundary.js` | `<svelte:boundary>` | 에러/서스펜스 바운더리 |
| `svelte-component.js` | `<svelte:component>` | 동적 컴포넌트 |
| `svelte-element.js` | `<svelte:element>` | 동적 HTML 엘리먼트 |
| `svelte-head.js` | `<svelte:head>` | `<head>` 내 콘텐츠 관리 |
| `html.js` | `{@html}` | raw HTML 삽입 |
| `css-props.js` | `--style-props` | CSS 커스텀 프로퍼티 전달 |
| `branches.js` | (내부) | if/each 등의 분기 관리 |
| `async.js` | (내부) | 비동기 블록 처리 |

## Elements

`elements/` 디렉토리가 HTML 엘리먼트의 세부 처리를 담당한다.

| 파일 | 역할 |
|------|------|
| `attributes.js` | HTML 어트리뷰트 설정/업데이트 |
| `bindings/` | `bind:` 디렉티브 (input, select, media 등) |
| `events.js` | 이벤트 핸들러 (`on:click` 등) |
| `transitions.js` | `transition:`, `in:`, `out:` |
| `class.js` | `class:` 디렉티브 |
| `style.js` | `style:` 디렉티브 |
| `actions.js` | `use:` 디렉티브 (액션) |
| `attachments.js` | `{@attach}` |
| `custom-element.js` | 커스텀 엘리먼트 지원 |
| `misc.js` | 기타 엘리먼트 처리 |

## Template System

컴파일러가 `.svelte` 파일을 JavaScript로 변환할 때, 정적인 HTML 구조를 `template()` 함수로 감싼다.

```
// 컴파일러 출력 예시
var root = template(`<div><span> </span></div>`);

function MyComponent($$anchor) {
    var fragment = root();       // template에서 DOM fragment 복제
    var div = get_first_child(fragment);
    var span = get_first_child(div);
    var text = get_first_child(span);

    // render_effect로 동적 부분만 업데이트
    render_effect(() => {
        set_text(text, count);
    });

    // anchor에 fragment 삽입
    append($$anchor, fragment);
}
```

> [!note] Virtual DOM이 아닌 이유
> Svelte는 컴파일 시점에 어떤 부분이 동적인지 알고 있으므로, 변경 가능한 부분에만 `render_effect`를 생성한다. diff 알고리즘 없이 직접 해당 DOM 노드를 업데이트한다.

## init_operations() (operations.js 31행)

DOM 조작 함수들을 lazy하게 초기화한다. 서버 환경에서는 `window`/`document`가 없으므로, 처음 사용할 때까지 초기화를 지연한다.

```
init_operations()
    |
    v
$window = window
$document = document
    |
    v
first_child_getter = Node.prototype.firstChild getter
next_sibling_getter = Node.prototype.nextSibling getter
    |
    v
Element.prototype에 __click, __className 등 확장
(DOM 조회 성능 최적화)
```

## Reconciler (each 블록)

`{#each}` 블록에서 리스트가 변경될 때, `reconciler.js`가 최소한의 DOM 조작으로 업데이트한다.

```
이전 리스트: [A, B, C, D]
새 리스트:   [A, C, D, E]

reconciler 결과:
  A: 유지 (위치 동일)
  B: 제거
  C: 이동 (index 2 -> 1)
  D: 이동 (index 3 -> 2)
  E: 생성 (새 항목)
```

`key` 속성이 있으면 키 기반 매칭, 없으면 인덱스 기반으로 처리한다.

## Hydration

SSR HTML에는 하이드레이션 마커가 포함된다:

```html
<!--[-->  (HYDRATION_START / BLOCK_OPEN)
<div>서버에서 렌더링된 콘텐츠</div>
<!--]-->  (HYDRATION_END / BLOCK_CLOSE)
```

하이드레이션 과정:

```
hydrate(component, { target })
    |
    v
+-------------------------+
| set_hydrating(true)     |  하이드레이션 모드 활성화
+-------------------------+
    |
    v
+-------------------------+
| 마커에서 시작 노드 탐색   |  <!--[--> 찾기
+-------------------------+
    |
    v
+-------------------------+
| 컴포넌트 함수 실행       |  DOM 생성 대신 기존 노드 재사용
| hydrate_node로 탐색     |  get_next_sibling로 이동
+-------------------------+
    |
    v
+-------------------------+
| 이벤트/반응성 연결       |  render_effect, 이벤트 바인딩
+-------------------------+
    |
    v
+-------------------------+
| set_hydrating(false)    |  하이드레이션 완료
+-------------------------+
```

## Block Rendering Flow

```
컴파일된 컴포넌트 함수
    |
    v
+---------------------+
| template() 호출      |  정적 DOM fragment 생성 (또는 하이드레이션 시 재사용)
+---------------------+
    |
    v
+---------------------+
| get_first_child()   |  DOM 트리 탐색으로 동적 노드 참조 획득
| get_next_sibling()  |
+---------------------+
    |
    v
+---------------------+
| Block 생성          |  if/each/await 등 제어 흐름
| (BRANCH_EFFECT)     |  조건에 따라 서브트리 생성/제거
+---------------------+
    |
    v
+---------------------+
| render_effect()     |  동적 값 바인딩
|                     |  (텍스트, 어트리뷰트, 클래스 등)
+---------------------+
    |
    v
+---------------------+
| append(anchor, dom) |  최종 DOM 삽입
+---------------------+
```

## Dependencies

- [[Client-Runtime]] - `runtime.js`의 `active_effect`, `get()` 등
- [[Reactivity-System]] - `render_effect`, `schedule_effect` 등

## 테스팅
- 실행: `pnpm test runtime-browser`, `pnpm test hydration`
- 패턴: 브라우저 기반 통합 테스트 (Playwright)
- 테스트 위치: `packages/svelte/tests/runtime-browser/`, `packages/svelte/tests/hydration/`

## Related Notes

- [[Client-Runtime]] - DOM을 사용하는 런타임 환경
- [[Reactivity-System]] - DOM 업데이트를 트리거하는 반응성 시스템
- [[Server-Runtime]] - 서버 측 DOM 대체 (문자열 렌더링)
- [[Practice-Client-Runtime]] - 연습 문제
