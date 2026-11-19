---
module: compiler-analyze-practice
path: packages/svelte/src/compiler/phases/2-analyze
keywords: analysis, visitor, scope, exercise, debug
tags: [module-analyze, practice]
---

# Practice: Analyze 단계 연습문제

> [!note] 학습 목표
> Analyze 단계의 visitor 패턴, 스코프 분석, CSS 분석, 유효성 검증의 동작을 이해합니다.

---

## Exercise 1 [trace]: visitor 패턴의 실행 흐름을 추적하세요

다음 코드가 분석될 때, 어떤 visitor들이 어떤 순서로 실행되는지 추적하세요:

```svelte
<script>
  let count = $state(0);
</script>

<button on:click={() => count++}>
  {count}
</button>
```

> [!answer]- 정답 보기
> `zimmerframe`의 `walk()` 함수가 AST를 깊이 우선(depth-first)으로 순회합니다:
>
> 1. **`_` (와일드카드)** - Root 노드에 대해 실행
> 2. **`Fragment`** - 최상위 프래그먼트
> 3. **`_`** -> **`VariableDeclarator`** - `count = $state(0)` 선언 분석
>    - 내부에서 `CallExpression` visitor 실행 -> `$state` rune 인식
>    - `count`가 `state` 바인딩으로 등록됨
> 4. **`_`** -> **`RegularElement`** - `<button>` 엘리먼트
>    - `visitors/shared/element.js`의 공통 로직 실행
>    - a11y 검사 실행
> 5. **`_`** -> **`OnDirective`** - `on:click` 디렉티브
>    - `ArrowFunctionExpression` -> `UpdateExpression`(`count++`)
>    - `count`의 `mutated` 플래그가 `true`로 설정됨
> 6. **`_`** -> **`ExpressionTag`** - `{count}`
>    - `Identifier` visitor -> `count` 변수 참조 해석
>    - 해당 바인딩의 참조 횟수 증가

---

## Exercise 2 [concept]: 스코프와 바인딩의 관계를 설명하세요

다음 코드에서 생성되는 스코프와 바인딩의 구조를 그려보세요:

```svelte
<script>
  let items = $state([1, 2, 3]);
  let doubled = $derived(items.map(x => x * 2));
</script>

{#each items as item, index}
  <p>{item} -> {doubled[index]}</p>
{/each}
```

> [!answer]- 정답 보기
> ```
> Module Scope (빈 상태 - <script context="module"> 없음)
>   |
>   +-- Instance Scope
>   |     |
>   |     +-- items: Binding { kind: 'state', mutated: false }
>   |     +-- doubled: Binding { kind: 'derived', mutated: false }
>   |
>   +-- EachBlock Scope
>         |
>         +-- item: Binding { kind: 'each', mutated: false }
>         +-- index: Binding { kind: 'each', mutated: false }
> ```
>
> **핵심 포인트:**
> - `$state()`와 `$derived()`는 `CallExpression` visitor에서 `get_rune()`으로 인식됨
> - `{#each}` 블록은 **새로운 스코프**를 생성하여 `item`, `index`를 로컬 변수로 등록
> - `doubled[index]`에서 `doubled`와 `index`는 서로 다른 스코프에 있지만, `Identifier` visitor가 스코프 체인을 탐색하여 올바른 바인딩을 찾음
> - 스코프 생성은 `compiler/scope.js`의 `create_scopes()` 함수에서 `walk()` 전에 처리됨

---

## Exercise 3 [debug]: 미사용 CSS 경고가 왜 발생하는지 분석하세요

다음 코드에서 CSS 경고가 발생합니다. 왜 발생하며, 분석 파이프라인에서 어디서 감지되는지 설명하세요:

```svelte
<div class="active">Hello</div>

<style>
  .active { color: green; }
  .inactive { color: red; }
</style>
```

> [!answer]- 정답 보기
> **`.inactive` 셀렉터가 미사용으로 감지됩니다.**
>
> **감지 과정:**
>
> 1. **`analyze_css()`** (`css/css-analyze.js`)
>    - CSS AST를 순회하며 각 셀렉터(`.active`, `.inactive`)를 추출
>    - 컴포넌트의 HTML AST와 대조하여 매칭 여부 확인
>    - `.active`는 `<div class="active">`와 매칭 -> 사용됨
>    - `.inactive`는 매칭되는 엘리먼트 없음 -> 미사용으로 마킹
>
> 2. **`prune()`** (`css/css-prune.js`)
>    - 미사용으로 마킹된 `.inactive` 규칙의 `metadata.used` 플래그를 `false`로 설정
>    - Transform 단계에서 이 규칙을 최종 CSS 출력에서 제거
>
> 3. **`warn_unused()`** (`css/css-warn.js`)
>    - `metadata.used === false`인 셀렉터에 대해 경고 생성
>    - `"Unused CSS selector '.inactive'"` 경고가 `state.warnings`에 추가됨
>
> **참고:** `/* svelte-ignore css_unused_selector */` 주석으로 경고를 억제할 수 있습니다.

---

## Exercise 4 [trace]: `$derived`가 분석되는 전체 과정을 추적하세요

```svelte
<script>
  let count = $state(0);
  let doubled = $derived(count * 2);
</script>
```

위 코드에서 `$derived(count * 2)`가 분석되는 과정을 상세히 설명하세요.

> [!answer]- 정답 보기
> 1. **스코프 생성 단계** (`create_scopes()`)
>    - `doubled` 변수가 Instance Scope에 등록됨
>    - 초기에는 일반 `let` 선언으로 처리
>
> 2. **`VariableDeclarator` visitor** 실행
>    - 초기화 표현식이 `CallExpression`인지 확인
>    - `get_rune()`으로 callee가 `$derived`인지 식별
>    - `$derived` rune으로 인식되면:
>      - `doubled`의 바인딩 `kind`를 `'derived'`로 변경
>      - 인자 `count * 2`를 표현식으로 기록
>
> 3. **`CallExpression` visitor** 실행
>    - `$derived`가 올바른 위치에서 호출되었는지 검증
>    - 인자 개수 확인 (1개여야 함)
>    - 모듈 스코프가 아닌 인스턴스 스코프에서만 허용
>
> 4. **`Identifier` visitor** - `count` 참조 분석
>    - `count`의 바인딩을 스코프 체인에서 찾음
>    - `count`가 `state` 바인딩임을 확인
>    - `doubled`가 `count`에 의존함을 의존성 그래프에 기록
>
> 5. **순환 의존성 검사** (`check_graph_for_cycles.js`)
>    - `count -> doubled` 의존성이 순환하지 않는지 확인
>    - 순환이 있으면 컴파일 에러 발생

---

## Exercise 5 [extend]: 새로운 visitor를 추가하는 방법을 설명하세요

새로운 AST 노드 타입 `MatchBlock`에 대한 분석 visitor를 추가한다고 가정할 때, 필요한 작업을 설명하세요.

> [!answer]- 정답 보기
> **1. Visitor 파일 생성**
> - `phases/2-analyze/visitors/MatchBlock.js` 파일 생성
> - 기존 visitor (예: `IfBlock.js`)를 참고하여 구조 작성:
>
> ```javascript
> /** @param {AST.MatchBlock} node */
> /** @param {Context} context */
> export function MatchBlock(node, context) {
>   // 표현식 분석
>   context.visit(node.expression);
>   // 각 case의 자식 노드 순회
>   for (const branch of node.cases) {
>     context.visit(branch);
>   }
> }
> ```
>
> **2. index.js에 등록**
> - `phases/2-analyze/index.js`에서 import 추가:
>   ```javascript
>   import { MatchBlock } from './visitors/MatchBlock.js';
>   ```
> - `visitors` 객체에 추가:
>   ```javascript
>   const visitors = { ..., MatchBlock, ... };
>   ```
>
> **3. 스코프 처리**
> - `MatchBlock`이 새로운 스코프를 생성해야 하면 `compiler/scope.js`의 `create_scopes()`에 처리 추가
>
> **4. 타입 정의**
> - `types.d.ts`의 `Visitors` 타입에 `MatchBlock` 추가
>
> **5. 유효성 검증**
> - visitor 내부에서 필요한 검증 로직 구현 (예: 중복 case 검사)

---

## Exercise 6 [debug]: 접근성(a11y) 경고는 어디서 어떻게 생성되나요?

```svelte
<img src="photo.jpg">
```

위 코드가 분석될 때 접근성 경고가 발생하는 과정을 추적하세요.

> [!answer]- 정답 보기
> 1. **`RegularElement` visitor** 실행
>    - `<img>` 태그를 분석할 때 `visitors/shared/element.js`의 공통 로직 호출
>
> 2. **`a11y/index.js`** 호출
>    - 엘리먼트 태그명이 `img`인지 확인
>    - `alt` 속성이 있는지 검사
>    - `alt` 속성이 없으면 `w.a11y_missing_attribute()` 경고 생성
>
> 3. **`a11y/constants.js`**
>    - `img` 태그에 필요한 속성 목록(`alt`)이 정의됨
>    - ARIA role 매핑, 인터랙티브 엘리먼트 목록 등 상수 데이터 포함
>
> 4. **경고 출력**
>    - `warnings.js`의 경고 함수가 호출됨
>    - `state.warnings` 배열에 경고 추가
>    - 최종 `CompileResult.warnings`에 포함되어 사용자에게 전달
>
> **경고 억제:** `<!-- svelte-ignore a11y_missing_attribute -->`를 `<img>` 앞에 추가

---

## Related Concepts

- [[Compiler-Analyze]] - Analyze 단계 상세 설명
- [[Compiler-Parse]] - AST를 생성하는 이전 단계
- [[Compiler-Transform]] - 코드 생성 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인
