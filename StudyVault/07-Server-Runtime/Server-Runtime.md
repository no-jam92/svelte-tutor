---
module: server-runtime
path: packages/svelte/src/internal/server
keywords: SSR, server, renderer, hydration, HTML, streaming
tags: [module-server, arch-runtime, pattern-dual-runtime]
---

# Server Runtime

> **Purpose**: 서버에서 Svelte 컴포넌트를 HTML 문자열로 렌더링하는 런타임. SvelteKit 등에서 SSR(Server-Side Rendering)을 수행할 때 사용된다.

> [!important] 중요도: ★★
> SSR의 핵심이지만, 클라이언트 런타임보다 구조가 단순하다. 반응성 시스템 없이 컴포넌트를 한 번 실행하여 HTML을 생성한다.

## Key Files

| 파일 | 역할 | 핵심 export |
|------|------|-------------|
| `index.js` | SSR 공개 API, 유틸리티 함수 | `render()`, `element()`, `attr()` |
| `renderer.js` | Renderer 클래스 (HTML 트리 구성) | `Renderer` |
| `context.js` | 서버 측 컴포넌트 컨텍스트 | `push()`, `pop()`, `ssr_context` |
| `hydration.js` | 하이드레이션 마커 상수 | `BLOCK_OPEN`, `BLOCK_CLOSE`, `EMPTY_COMMENT` |
| `dev.js` | 개발 모드 전용 기능 | 서버 측 디버깅 |
| `errors.js` | 서버 런타임 에러 | 에러 헬퍼 함수 |
| `render-context.js` | 렌더 컨텍스트 관리 | `get_render_context()`, `with_render_context()` |
| `crypto.js` | CSP용 해시 생성 | `sha256()` |
| `abort-signal.js` | 렌더링 중단 시그널 | `abort()` |

### Server Blocks (`blocks/`)

서버에서도 클라이언트의 블록 구조에 대응하는 렌더링 로직이 있다.

## Renderer 클래스 (renderer.js)

`Renderer`는 HTML 출력을 트리 구조로 관리하는 핵심 클래스이다 (renderer.js 34행).

```
Renderer
  |
  +-- #out: RendererItem[]     // (string | Renderer)[] 콘텐츠
  +-- #on_destroy              // onDestroy 콜백
  +-- #is_component_body       // 컴포넌트 본문 여부
  +-- #boundary                // 에러 바운더리 설정
  +-- type: 'head' | 'body'    // 출력 타입 (head vs body)
  +-- #parent: Renderer        // 부모 렌더러
  +-- promise                  // 비동기 작업
  +-- global: SSRState         // 전역 상태 (트리 전체 공유)
  +-- local                    // 지역 상태 (분기별 복사)
```

> [!note] Renderer 트리 구조
> Renderer는 문자열과 자식 Renderer의 배열로 콘텐츠를 축적한다. 비동기 렌더링 시 자식 Renderer가 아직 완료되지 않았을 수 있으며, `await`로 비동기 수집(collect)할 수 있다.

## SSR Rendering Process

```
render(Component, options)
    |
    v
+------------------------+
| Renderer.render()      |  정적 메서드 (renderer.js)
+------------------------+
    |
    v
+------------------------+
| new Renderer(global)   |  루트 Renderer 생성
|   type: 'body'         |
+------------------------+
    |
    v
+------------------------+
| push(context)          |  서버 컨텍스트 설정
| set_ssr_context()      |
+------------------------+
    |
    v
+------------------------+
| Component(renderer, props) | 컴포넌트 함수 실행
|                        |  renderer.push(string)으로
|                        |  HTML 문자열 축적
+------------------------+
    |
    v
+------------------------+
| 하위 컴포넌트 재귀 실행  |  자식도 동일한 renderer에
|                        |  또는 child renderer에 출력
+------------------------+
    |
    v
+------------------------+
| pop()                  |  컨텍스트 정리
+------------------------+
    |
    v
+------------------------+
| collect()              |  Renderer 트리 -> 문자열
+------------------------+
    |
    v
{ body: string, head: string }
```

## render() 함수 (index.js 71행)

```javascript
export function render(component, options = {}) {
    if (options.csp?.hash && options.csp.nonce) {
        e.invalid_csp();  // hash와 nonce는 동시 사용 불가
    }
    return Renderer.render(component, options);
}
```

반환값은 `{ body: string, head: string }` 형태의 `RenderOutput`이다.

### Options

- `props`: 컴포넌트 props
- `context`: `Map<any, any>` 형태의 초기 컨텍스트
- `idPrefix`: 하이드레이션 ID 접두사
- `csp`: Content Security Policy (`hash` 또는 `nonce`)
- `transformError`: 에러 변환 함수

## element() 함수 (index.js 40행)

동적 엘리먼트 (`<svelte:element>`) 서버 렌더링:

```javascript
export function element(renderer, tag, attributes_fn, children_fn) {
    renderer.push('<!---->');       // 블록 마커
    if (tag) {
        renderer.push(`<${tag}`);
        attributes_fn();            // 어트리뷰트 출력
        renderer.push(`>`);
        if (!is_void(tag)) {
            children_fn();           // 자식 콘텐츠 출력
            renderer.push(`</${tag}>`);
        }
    }
    renderer.push('<!---->');       // 블록 마커
}
```

## Hydration Markers

서버 출력에 하이드레이션 마커를 삽입하여 클라이언트의 `hydrate()`가 DOM 구조를 파악할 수 있게 한다.

| 마커 | 상수 | 용도 |
|------|------|------|
| `<!--[-->` | `BLOCK_OPEN` | 블록 시작 |
| `<!--]-->` | `BLOCK_CLOSE` | 블록 끝 |
| `<!---->` | `EMPTY_COMMENT` | 빈 블록/동적 엘리먼트 경계 |

## 클라이언트 런타임과의 차이

| 특성 | Client Runtime | Server Runtime |
|------|---------------|----------------|
| 반응성 | 시그널 기반 (source, effect) | 없음 (한 번 실행) |
| DOM | 실제 DOM 조작 | HTML 문자열 축적 |
| 이벤트 | 이벤트 핸들러 등록 | 무시 |
| 라이프사이클 | mount, update, destroy | 실행 + onDestroy만 |
| 출력 | DOM 노드 | `{ body, head }` 문자열 |
| Proxy | 객체/배열 반응성 | 불필요 |
| 비동기 | Batch, microtask | Renderer의 promise 기반 |
| 에러 처리 | 에러 바운더리 (DOM) | Renderer의 #boundary |

## Streaming Support

`Renderer`의 `promise` 프로퍼티와 비동기 수집 패턴을 통해 비동기 렌더링을 지원한다. 자식 Renderer가 비동기 작업을 수행하면 `await renderer`로 완료를 기다릴 수 있다. 이는 SvelteKit의 스트리밍 SSR과 연동된다.

## Dependencies

- `../../constants.js` - `HYDRATION_START`, `HYDRATION_END` 등
- `../../escaping.js` - HTML 이스케이프 (`escape_html`)
- `../shared/attributes.js` - `attr()`, `clsx()`, `to_class()`, `to_style()`
- `../../store/utils.js` - 스토어 구독 유틸리티
- `devalue` - 직렬화 라이브러리

## Used by

- SvelteKit의 SSR 파이프라인
- `svelte/server`의 `render()` 공개 API

## 테스팅
- 실행: `pnpm test server-side-rendering`
- 패턴: 단위 테스트 (Vitest)
- 테스트 위치: `packages/svelte/tests/server-side-rendering/`

## Related Notes

- [[Client-Runtime]] - 클라이언트 측 대응 모듈
- [[DOM-Operations]] - 하이드레이션 시 서버 출력과 연결되는 DOM 레이어
- [[Reactivity-System]] - 서버에서는 사용되지 않지만 개념적으로 연관
- [[Practice-Server-Runtime]] - 연습 문제
