---
module: compiler-transform-practice
path: packages/svelte/src/compiler/phases/3-transform
keywords: transform, codegen, exercise, client, server, esrap
tags: [module-transform, practice]
---

# Practice: Transform 단계 연습문제

> [!note] 학습 목표
> Transform 단계의 클라이언트/서버 변환, 코드 생성, CSS 스코핑의 동작을 이해합니다.

---

## Exercise 1 [trace]: `$state` 변수가 클라이언트 코드로 변환되는 과정을 추적하세요

```svelte
<script>
  let count = $state(0);
</script>

<p>{count}</p>
```

위 코드가 클라이언트용 JavaScript로 변환되는 과정을 추적하세요.

> [!answer]- 정답 보기
> 1. **`transform_component()` 호출** (`index.js`)
>    - `options.generate`가 `'client'`(기본값)이므로 `client_component()` 호출
>
> 2. **`client/transform-client.js`의 `client_component()`**
>    - visitor 객체를 구성하고 `walk()`으로 AST 순회 시작
>    - `Program` visitor가 모듈 래퍼 코드 생성
>
> 3. **`VariableDeclaration` visitor** (`client/visitors/VariableDeclaration.js`)
>    - `count`의 바인딩이 `kind: 'state'`인 것을 확인
>    - `let count = $state(0);` -> `let count = $.state(0);` 런타임 호출로 변환
>    - (내부적으로 `$.source(0)` 또는 유사한 시그널 생성 코드로 변환)
>
> 4. **`Fragment` visitor** (`client/visitors/Fragment.js`)
>    - `<p>{count}</p>` 템플릿 처리
>    - `transform-template/`에서 정적 HTML 부분을 템플릿 문자열로 최적화
>    - `$.template("<p> </p>")` 형태의 템플릿 생성
>
> 5. **`ExpressionTag`에서 `Identifier` visitor**
>    - `count` 참조를 `$.get(count)` 형태의 시그널 읽기로 변환
>    - `$.text()`로 텍스트 노드 업데이트 코드 생성
>
> 6. **`esrap`의 `print()`** 로 ESTree AST -> JavaScript 소스 코드 생성
>
> **최종 출력 (간략화):**
> ```javascript
> import * as $ from 'svelte/internal/client';
> export default function Component($$anchor) {
>   let count = $.state(0);
>   var fragment = $.template("<p> </p>");
>   var p = $.first_child(fragment);
>   var text = $.child(p);
>   $.template_effect(() => $.set_text(text, $.get(count)));
>   $.append($$anchor, fragment);
> }
> ```

---

## Exercise 2 [compare]: 같은 코드의 클라이언트 vs 서버 변환 결과를 비교하세요

```svelte
<script>
  let name = $state("World");
</script>

<h1>Hello {name}!</h1>
```

이 코드가 클라이언트와 서버에서 각각 어떻게 변환되는지 비교하세요.

> [!answer]- 정답 보기
> **Client Transform** (`client_component()`):
> ```javascript
> // DOM 조작 기반
> import * as $ from 'svelte/internal/client';
> export default function Component($$anchor) {
>   let name = $.state("World");
>   var fragment = $.template("<h1>Hello !</h1>");
>   var h1 = $.first_child(fragment);
>   var text = $.child(h1, 1); // "Hello " 다음의 텍스트 노드
>   $.template_effect(() => $.set_text(text, $.get(name)));
>   $.append($$anchor, fragment);
> }
> ```
>
> **Server Transform** (`server_component()`):
> ```javascript
> // 문자열 연결 기반
> import * as $ from 'svelte/internal/server';
> export default function Component($$payload) {
>   let name = "World";  // 서버에서는 시그널 불필요
>   $$payload.out += `<h1>Hello ${$.escape(name)}!</h1>`;
> }
> ```
>
> **주요 차이점:**
>
> | 항목 | Client | Server |
> |------|--------|--------|
> | 렌더링 방식 | DOM 조작 | 문자열 연결 |
> | 반응성 | `$.state()` 시그널 | 일반 변수 |
> | 이스케이프 | 불필요 (DOM API) | `$.escape()` 필수 (XSS 방지) |
> | 이벤트 | 바인딩됨 | 생략됨 |
> | 출력 크기 | 더 큼 (런타임 코드) | 더 작음 (단순 문자열) |

---

## Exercise 3 [trace]: CSS 스코핑이 적용되는 과정을 추적하세요

```svelte
<div class="card">
  <p>Content</p>
</div>

<style>
  .card { background: white; }
  .card p { color: black; }
</style>
```

CSS가 스코핑되어 최종 출력에 포함되는 과정을 설명하세요.

> [!answer]- 정답 보기
> 1. **Analyze 단계에서**
>    - `analyze_css()`: `.card`와 `.card p` 셀렉터가 컴포넌트 내 엘리먼트와 매칭되는지 분석
>    - 둘 다 사용됨으로 마킹
>    - 컴포넌트 해시 생성: `hash()` 함수로 소스 코드 기반 고유 해시 계산 (예: `svelte-abc123`)
>
> 2. **Transform 단계 - `render_stylesheet()`** (`css/index.js`)
>    - 각 CSS 셀렉터에 해시를 추가하여 스코핑:
>      ```css
>      /* 변환 전 */
>      .card { background: white; }
>      .card p { color: black; }
>
>      /* 변환 후 */
>      .card.svelte-abc123 { background: white; }
>      .card.svelte-abc123 p.svelte-abc123 { color: black; }
>      ```
>    - 미사용으로 마킹된 규칙은 출력에서 제외
>    - CSS 소스맵 생성
>
> 3. **HTML 엘리먼트에도 해시 클래스 추가**
>    - 클라이언트 변환에서 `<div class="card">` -> `<div class="card svelte-abc123">`
>    - `<p>` 태그에도 `svelte-abc123` 클래스 추가
>
> 4. **최종 출력**
>    - `CompileResult.css.code` - 스코핑된 CSS 코드
>    - `CompileResult.css.map` - CSS 소스맵
>    - `options.css()`가 `'injected'`이면 JS 번들에 CSS가 인라인으로 포함

---

## Exercise 4 [debug]: 소스맵이 올바르지 않을 때 어디를 확인해야 하나요?

변환된 코드의 소스맵이 원본 `.svelte` 파일과 올바르게 매핑되지 않을 때 디버깅 방법을 설명하세요.

> [!answer]- 정답 보기
> **소스맵 생성 경로:**
>
> 1. **`esrap`의 `print()` 함수** (`index.js`)
>    - AST 노드의 `start`/`end` 위치를 기반으로 소스맵 생성
>    - `sourceMapContent`와 `sourceMapSource` 옵션으로 원본 소스 정보 전달
>    - **확인 포인트:** 변환된 AST 노드들의 `start`/`end` 값이 원본 소스의 올바른 위치를 가리키는지
>
> 2. **`merge_with_preprocessor_map()`** (`utils/mapped_code.js`)
>    - 전처리기(svelte-preprocess 등)가 소스맵을 생성한 경우 이와 병합
>    - **확인 포인트:** 전처리기 소스맵이 유효한지, 병합 과정에서 오프셋이 맞는지
>
> 3. **`get_source_name()`**
>    - 소스맵에 기록되는 파일명 결정
>    - `options.filename`과 `options.outputFilename` 기반
>
> **디버깅 순서:**
> 1. `CompileResult.js.map`을 JSON으로 출력하여 `mappings` 필드 확인
> 2. [source-map-visualization](https://evanw.github.io/source-map-visualization/) 등 도구로 시각화
> 3. 문제가 되는 줄의 AST 노드를 찾아 `start`/`end` 값이 올바른지 확인
> 4. 전처리기를 사용하는 경우, 전처리기 출력의 소스맵을 별도 검증
> 5. `esrap`의 `print()` 결과와 병합 후 결과를 각각 비교

---

## Exercise 5 [extend]: 새로운 런타임 함수를 생성하는 visitor를 추가하는 방법

가상의 `{#match}` 블록을 클라이언트 코드로 변환하는 visitor를 추가한다고 할 때, 필요한 작업을 설명하세요.

> [!answer]- 정답 보기
> **1. Client Visitor 생성**
> - `client/visitors/MatchBlock.js` 파일 생성
> - 기존 `IfBlock.js`를 참고하여 구조 작성:
>
> ```javascript
> /** @param {AST.MatchBlock} node */
> export function MatchBlock(node, context) {
>   // 1. 매치 표현식 변환
>   const expression = context.visit(node.expression);
>
>   // 2. 각 case 브랜치를 함수로 변환
>   const cases = node.cases.map(c => {
>     const body = context.visit(c.body);
>     return b.arrow_function([], body);
>   });
>
>   // 3. 런타임 함수 호출 생성
>   // $.match(expression, [case1_fn, case2_fn, ...])
>   return b.call('$.match', expression, b.array(cases));
> }
> ```
>
> **2. Server Visitor 생성**
> - `server/visitors/MatchBlock.js` - 서버용 문자열 렌더링 로직
>
> **3. visitor 등록**
> - `client/transform-client.js`에서 visitor 객체에 `MatchBlock` 추가
> - `server/transform-server.js`에서도 동일하게 추가
>
> **4. 런타임 함수 구현**
> - `packages/svelte/src/internal/client/`에 `$.match()` 런타임 함수 구현
> - `packages/svelte/src/internal/server/`에도 서버 버전 구현
>
> **5. 빌더 함수** (`compiler/builders.js`)
> - 필요한 경우 AST 노드 생성 헬퍼 추가
>
> **핵심:** Transform visitor는 Svelte AST 노드를 **ESTree AST 노드**로 변환합니다. 이 ESTree 노드는 이후 esrap이 JavaScript 코드로 출력합니다.

---

## Exercise 6 [trace]: `transform-template`은 어떤 최적화를 수행하나요?

```svelte
<div class="container">
  <h1>Title</h1>
  <p>Static text</p>
  <p>{dynamicValue}</p>
</div>
```

위 코드에서 정적 부분과 동적 부분이 어떻게 분리되어 최적화되는지 설명하세요.

> [!answer]- 정답 보기
> **`transform-template/` 디렉토리의 역할:**
>
> 정적 HTML을 **템플릿 문자열**로 추출하여 런타임에 `innerHTML`이나 `template` 엘리먼트로 효율적으로 생성합니다.
>
> **분리 과정:**
>
> 1. **정적 부분 추출**
>    - `<div class="container">`, `<h1>Title</h1>`, `<p>Static text</p>` 는 변하지 않는 정적 HTML
>    - `template.js`에서 이들을 하나의 템플릿 문자열로 합침:
>      ```javascript
>      var root = $.template(
>        '<div class="container"><h1>Title</h1><p>Static text</p><p> </p></div>'
>      );
>      ```
>    - `<p> </p>`의 공백은 동적 텍스트가 들어갈 자리표시자
>
> 2. **동적 부분 바인딩**
>    - 템플릿에서 동적 노드의 위치를 찾는 코드 생성:
>      ```javascript
>      var div = $.first_child(fragment);
>      var p2 = $.sibling($.sibling($.first_child(div), 2), 2); // 세 번째 자식
>      var text = $.child(p2);
>      ```
>    - 이펙트로 동적 값 업데이트:
>      ```javascript
>      $.template_effect(() => $.set_text(text, dynamicValue));
>      ```
>
> 3. **최적화 이점**
>    - 정적 HTML은 한 번만 파싱 (`$.template()`이 내부적으로 캐싱)
>    - `createElement` 반복 호출 대신 `cloneNode(true)`로 빠르게 복제
>    - 동적 부분만 이펙트로 업데이트하여 최소한의 DOM 조작
>
> **`fix-attribute-casing.js`** 는 SVG/MathML에서 대소문자가 중요한 속성(예: `viewBox`, `preserveAspectRatio`)을 올바르게 보정합니다.

---

## Related Concepts

- [[Compiler-Transform]] - Transform 단계 상세 설명
- [[Compiler-Analyze]] - 분석 결과를 입력으로 받는 이전 단계
- [[Compiler-Parse]] - 파싱 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인
