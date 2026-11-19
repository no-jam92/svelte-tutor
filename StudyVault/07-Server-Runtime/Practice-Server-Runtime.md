---
module: server-runtime
type: practice
keywords: SSR, server, renderer, hydration, exercises
tags: [module-server, practice]
---

# Practice: Server Runtime

> [!tip] 학습 방법
> SSR의 전체 흐름을 이해하려면 서버에서 렌더링된 HTML 출력을 직접 확인하고, 클라이언트에서 하이드레이션되는 과정과 연결해서 생각하세요.

## Q1. [trace] render() 호출부터 HTML 문자열 반환까지의 전체 흐름을 추적하세요

```javascript
import { render } from 'svelte/server';
import App from './App.svelte';

const { body, head } = render(App, { props: { name: 'world' } });
```

> [!answer]- 정답 보기
> 1. **`render()` 호출** (index.js 71행): CSP 옵션 유효성 검사 후 `Renderer.render()` 정적 메서드 호출.
> 2. **`Renderer.render()`** (renderer.js): 새 `SSRState` global 객체 생성, 루트 `Renderer` 인스턴스 생성 (type: 'body').
> 3. **컨텍스트 설정**: `push()`로 컴포넌트 컨텍스트 스택 push, `set_ssr_context()`로 SSR 컨텍스트 설정. options.context가 있으면 초기 컨텍스트로 설정.
> 4. **컴포넌트 함수 실행**: `App(renderer, props)` 호출. 컴파일러가 서버용으로 생성한 코드는 `renderer.push(string)`으로 HTML 축적.
> 5. **하위 컴포넌트**: 자식 컴포넌트도 같은 renderer (또는 `renderer.child()`)에 HTML 출력. `<!--[-->` / `<!--]-->` 마커 삽입.
> 6. **`<svelte:head>` 처리**: `renderer.child('head')`로 type을 'head'로 전환한 자식 renderer에 출력.
> 7. **`pop()`**: 컨텍스트 정리, `onDestroy` 콜백 실행.
> 8. **`collect()`**: Renderer 트리를 순회하며 모든 문자열을 연결. type별로 분리하여 `{ body, head }` 반환.

## Q2. [compare] 같은 컴포넌트의 클라이언트 코드와 서버 코드가 어떻게 다른가요?

컴파일러가 같은 `.svelte` 파일에서 클라이언트용과 서버용 코드를 어떻게 다르게 생성하는지 비교하세요.

> [!answer]- 정답 보기
> 같은 컴포넌트:
> ```svelte
> <script>
>     let count = $state(0);
> </script>
> <button onclick={() => count++}>{count}</button>
> ```
>
> **클라이언트 코드** (개념적):
> ```javascript
> import { source, set } from 'svelte/internal/client';
> import { template, render_effect, set_text } from 'svelte/internal/client';
>
> var root = template(`<button> </button>`);
> function App($$anchor) {
>     let count = source(0);
>     var button = root();
>     var text = get_first_child(button);
>     button.__click = () => set(count, get(count) + 1);
>     render_effect(() => set_text(text, get(count)));
>     append($$anchor, button);
> }
> ```
>
> **서버 코드** (개념적):
> ```javascript
> function App($$renderer, $$props) {
>     let count = 0;  // 시그널 없이 평범한 변수
>     $$renderer.push(`<!--[--><button>`);
>     $$renderer.push(escape_html(count));
>     $$renderer.push(`</button><!--]-->`);
>     // 이벤트 핸들러는 완전히 무시됨
> }
> ```
>
> **핵심 차이:**
> - 서버: `$state` -> 일반 변수, 반응성 없음
> - 서버: 이벤트 핸들러 코드 제거
> - 서버: DOM API 대신 `renderer.push(string)`
> - 서버: `escape_html()`로 XSS 방지
> - 서버: 하이드레이션 마커 (`<!--[-->`, `<!--]-->`) 삽입

## Q3. [trace] 하이드레이션 마커가 어떻게 사용되는지 서버-클라이언트 전체 과정을 추적하세요

> [!answer]- 정답 보기
> **서버 측:**
> 1. 각 블록 (`{#if}`, `{#each}` 등)이 렌더링될 때 `renderer.push(BLOCK_OPEN)` (`<!--[-->`)으로 시작.
> 2. 블록 내용 렌더링.
> 3. `renderer.push(BLOCK_CLOSE)` (`<!--]-->`)로 끝.
> 4. 조건부 블록에서 else 분기는 `BLOCK_OPEN_ELSE` 사용.
>
> **생성된 HTML 예시:**
> ```html
> <!--[--><div><!--[-->항목 1<!--]--><!--[-->항목 2<!--]--><!--[-->항목 3<!--]--></div><!--]-->
> ```
>
> **클라이언트 측 (hydrate()):**
> 1. `hydrate()` (render.js 96행)가 `set_hydrating(true)` 설정.
> 2. 컴포넌트 함수 실행 시, `template()`이 새 DOM을 만드는 대신 `hydrate_node`에서 시작.
> 3. 각 블록이 `<!--[-->` 마커를 찾아 자신의 시작 지점 확인.
> 4. `get_next_sibling()`로 실제 콘텐츠 노드를 탐색하며 이벤트/반응성 연결.
> 5. `<!--]-->` 마커에서 블록 종료를 확인.
> 6. 마커 사이의 노드 수를 통해 서버/클라이언트 렌더링 결과 불일치 감지 가능.

## Q4. [debug] SSR에서 에러가 발생했을 때 디버깅 방법은?

서버에서 컴포넌트 렌더링 중 에러가 발생하는 경우의 디버깅 전략을 설명하세요.

> [!answer]- 정답 보기
> **디버깅 전략:**
>
> 1. **`<svelte:boundary>` 확인**: Renderer의 `#boundary` (renderer.js 56-61행)가 에러 바운더리 역할. `failed` 스니펫이 설정되어 있으면 에러 시 대체 콘텐츠를 렌더링.
>
> 2. **`transformError` 옵션**: `render()` 호출 시 `transformError` 함수를 전달하여 에러를 변환/로깅할 수 있다.
>
> 3. **컨텍스트 관련 에러**: `getContext()`가 서버에서 실패하면 `ssr_context` (context.js)가 올바르게 설정되었는지 확인. 서버에서는 `push()`/`pop()`이 다른 구현을 사용.
>
> 4. **비동기 에러**: Renderer의 `promise`가 reject되면 비동기 컴포넌트에서 에러 발생. `await` 구문이 서버에서 올바르게 처리되는지 확인.
>
> 5. **브라우저 전용 API**: `window`, `document`, `localStorage` 등을 서버 코드에서 접근하면 에러. `$effect`는 서버에서 실행되지 않으므로 안전하지만, 모듈 최상위 코드에서 브라우저 API를 사용하면 문제.
>
> 6. **dev 모드**: `dev.js`가 서버에서도 추가 검증을 수행. `DEV` 플래그로 활성화.

## Q5. [extend] CSP(Content Security Policy)가 SSR에서 어떻게 처리되나요?

> [!answer]- 정답 보기
> `render()` 옵션에서 `csp` 객체를 전달할 수 있다 (index.js 68행):
>
> ```javascript
> render(App, {
>     csp: { hash: (content) => addToHash(content) }
>     // 또는
>     csp: { nonce: 'abc123' }
> });
> ```
>
> - **`hash` 모드**: 인라인 스크립트/스타일의 해시를 수집. `crypto.js`의 `sha256()`으로 해시 생성. CSP 헤더에 `sha256-...` 형식으로 추가.
> - **`nonce` 모드**: 인라인 스크립트/스타일에 `nonce="abc123"` 속성 추가.
> - **동시 사용 불가**: `hash`와 `nonce`를 동시 지정하면 `e.invalid_csp()` 에러 (index.js 72-74행).
>
> Renderer의 `global` SSRState가 CSP 정보를 트리 전체에서 공유하여, 모든 하위 컴포넌트의 인라인 콘텐츠에 일관되게 적용한다.

## Q6. [trace] Renderer의 비동기 렌더링은 어떻게 동작하나요?

> [!answer]- 정답 보기
> Renderer는 비동기 콘텐츠를 트리 구조로 관리한다:
>
> 1. **`#out` 배열**: `string | Renderer` 형태. 동기 콘텐츠는 문자열로, 비동기 콘텐츠는 자식 Renderer로 저장.
>
> 2. **자식 Renderer 생성**: `{#await promise}` 같은 비동기 블록에서 `renderer.child()`로 새 Renderer 생성. 이 Renderer의 `promise` 프로퍼티에 비동기 작업이 할당됨.
>
> 3. **동기 수집 (`collect`)**: `#out`을 순회하며 문자열을 연결. 자식 Renderer가 아직 완료되지 않은 `promise`를 가지면 에러 발생.
>
> 4. **비동기 수집 (`await renderer`)**: 모든 자식의 `promise`가 완료될 때까지 기다린 후 문자열 수집.
>
> 5. **에러 바운더리와 연동**: `#boundary` 설정이 있으면, 자식의 비동기 수집 실패 시 `failed` 스니펫으로 대체 렌더링.
>
> 이 구조 덕분에 SvelteKit은 스트리밍 SSR에서 즉시 렌더링 가능한 부분을 먼저 전송하고, 비동기 부분은 완료 후 전송할 수 있다.

## Related Notes

- [[Server-Runtime]] - 이 문제들의 배경 지식
- [[Client-Runtime]] - 클라이언트 측 대응 모듈
- [[DOM-Operations]] - 하이드레이션과의 연결
- [[Practice-Client-Runtime]] - 클라이언트 런타임 연습 문제
