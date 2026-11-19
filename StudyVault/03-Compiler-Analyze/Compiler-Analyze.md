---
module: compiler-analyze
path: packages/svelte/src/compiler/phases/2-analyze
keywords: analysis, visitor, scope, binding, validation, zimmerframe
tags: [module-analyze, arch-compiler, pattern-visitor]
---

# Compiler Analyze (2-analyze)

> **Purpose:** AST를 분석하여 스코프, 바인딩, 의존성을 해석하는 단계
> **Importance:** ★★★

## Overview

Analyze 단계는 Parse 단계에서 생성된 AST를 입력받아, **스코프 분석**, **변수 바인딩**, **의존성 추적**, **유효성 검증**, **CSS 분석**을 수행합니다. 이 단계의 결과는 풍부한 메타데이터가 추가된 **enriched AST**로, [[Compiler-Transform]] 단계가 정확한 코드를 생성하는 데 필수적입니다.

## Key Files

### 최상위 파일

| 파일 | 역할 |
|------|------|
| `index.js` | `analyze_component()`, `analyze_module()` 내보내기 (진입점) |
| `types.d.ts` | `AnalysisState`, `Visitors` 등 타입 정의 |

### Visitor 파일 (visitors/)

분석 단계의 핵심 로직은 각 AST 노드 타입별 visitor 파일에 분산되어 있습니다:

| 카테고리 | Visitor 파일들 |
|----------|---------------|
| **블록 구문** | `IfBlock.js`, `EachBlock.js`, `AwaitBlock.js`, `KeyBlock.js`, `SnippetBlock.js` |
| **컴포넌트** | `Component.js`, `SvelteComponent.js`, `SvelteSelf.js`, `SvelteElement.js` |
| **엘리먼트** | `RegularElement.js`, `SlotElement.js`, `TitleElement.js` |
| **특수 엘리먼트** | `SvelteBody.js`, `SvelteWindow.js`, `SvelteDocument.js`, `SvelteHead.js`, `SvelteBoundary.js`, `SvelteFragment.js` |
| **디렉티브** | `BindDirective.js`, `OnDirective.js`, `UseDirective.js`, `TransitionDirective.js`, `AnimateDirective.js`, `ClassDirective.js`, `StyleDirective.js`, `LetDirective.js` |
| **표현식** | `Identifier.js`, `MemberExpression.js`, `CallExpression.js`, `AssignmentExpression.js`, `UpdateExpression.js`, `AwaitExpression.js`, `NewExpression.js`, `TaggedTemplateExpression.js` |
| **선언문** | `VariableDeclarator.js`, `FunctionDeclaration.js`, `FunctionExpression.js`, `ArrowFunctionExpression.js`, `ClassDeclaration.js`, `ClassBody.js`, `PropertyDefinition.js` |
| **모듈** | `ImportDeclaration.js`, `ExportNamedDeclaration.js`, `ExportDefaultDeclaration.js`, `ExportSpecifier.js` |
| **템플릿 태그** | `ExpressionTag.js`, `HtmlTag.js`, `ConstTag.js`, `DebugTag.js`, `RenderTag.js`, `AttachTag.js` |
| **기타** | `Fragment.js`, `Text.js`, `Attribute.js`, `SpreadAttribute.js`, `SpreadElement.js`, `Literal.js`, `TemplateElement.js`, `LabeledStatement.js`, `ExpressionStatement.js` |

### Shared 유틸리티 (visitors/shared/)

| 파일 | 역할 |
|------|------|
| `element.js` | 엘리먼트 공통 분석 로직 |
| `component.js` | 컴포넌트 공통 분석 로직 |
| `attribute.js` | 속성 분석 공통 로직 |
| `fragment.js` | 프래그먼트 동적 마킹 (`mark_subtree_dynamic`) |
| `function.js` | 함수 관련 공통 분석 |
| `snippets.js` | 스니펫 분석 |
| `special-element.js` | 특수 엘리먼트 (`<svelte:*>`) 자식 요소 검증 |
| `utils.js` | 분석 유틸리티 함수 |
| `a11y/index.js` | 접근성(a11y) 검사 |
| `a11y/constants.js` | 접근성 관련 상수 |

### CSS 분석 (css/)

| 파일 | 역할 |
|------|------|
| `css-analyze.js` | CSS AST 분석 (`analyze_css()`) |
| `css-prune.js` | 사용되지 않는 CSS 규칙 제거 (`prune()`) |
| `css-warn.js` | 미사용 CSS 셀렉터 경고 (`warn_unused()`) |
| `utils.js` | CSS 분석 유틸리티 |

### 유틸리티 (utils/)

| 파일 | 역할 |
|------|------|
| `check_graph_for_cycles.js` | 반응형 선언 간 순환 의존성 감지 |

## Visitor Pattern 설명

Analyze 단계는 **Visitor 패턴**을 사용합니다. `zimmerframe`의 `walk()` 함수로 AST를 순회하며, 각 노드 타입에 매칭되는 visitor 함수를 실행합니다.

```javascript
// index.js에서 visitor 객체 구성 예시
const visitors = {
  // 모든 노드에 대해 실행되는 기본 visitor
  _(node, { state, next, path }) {
    // svelte-ignore 주석 처리
    // 다음 자식 노드로 순회 계속
    next();
  },

  // 특정 노드 타입별 visitor
  Identifier(node, context) { /* 변수 참조 분석 */ },
  BindDirective(node, context) { /* 바인딩 유효성 검사 */ },
  Component(node, context) { /* 컴포넌트 분석 */ },
  // ... 60개 이상의 visitor
};

// zimmerframe으로 AST 순회
walk(ast, state, visitors);
```

> [!important] Visitor 실행 순서
> `_` (와일드카드) visitor가 **모든** 노드에 대해 먼저 실행된 후, 해당 노드 타입의 구체적인 visitor가 실행됩니다.

## Scope Analysis (스코프 분석)

스코프 분석은 `compiler/scope.js`에서 처리됩니다:

```
+---------------------------+
| ScopeRoot                 |
|  +----------------------+ |
|  | Module Scope         | |  <-- <script context="module">의 변수들
|  |  +------------------+| |
|  |  | Instance Scope   || |  <-- <script>의 변수들 ($state, $derived 등)
|  |  |  +--------------+|| |
|  |  |  | Block Scope  ||| |  <-- {#each}, {#if} 등의 로컬 변수
|  |  |  +--------------+|| |
|  |  +------------------+| |
|  +----------------------+ |
+---------------------------+
```

- **`create_scopes()`** - AST를 순회하며 스코프 트리 구축
- **`Scope` 클래스** - 변수 선언, 참조, 바인딩 정보 관리
- **`get_rune()`** - `$state`, `$derived`, `$effect` 등 Svelte 5 rune 식별
- **`Binding` 객체** - 각 변수의 종류(prop, state, derived, let 등), 변이 여부, 참조 횟수 추적

## CSS Analysis (CSS 분석)

CSS 분석은 세 단계로 이루어집니다:

1. **`analyze_css()`** - CSS 셀렉터가 컴포넌트 내 어떤 엘리먼트에 매칭되는지 분석
2. **`prune()`** - 매칭되지 않는 CSS 규칙을 `unused`로 마킹하여 제거 대상으로 표시
3. **`warn_unused()`** - 사용되지 않는 CSS 셀렉터에 대해 경고 출력

> [!tip] Svelte의 스코프드 CSS
> Svelte는 컴포넌트별로 CSS를 스코핑합니다. analyze 단계에서 각 CSS 셀렉터가 실제로 사용되는지 정적 분석하여, 불필요한 스타일을 제거합니다.

## Validation (유효성 검증)

각 visitor 내부에서 구조적 유효성을 검증합니다:

- **컴포넌트 구조**: `<script>`는 최대 2개 (instance + module), `<style>`은 최대 1개
- **디렉티브 호환성**: `bind:value`가 해당 엘리먼트에서 유효한지
- **Rune 사용법**: `$state()`가 올바른 위치에서 사용되었는지
- **접근성(a11y)**: `a11y/index.js`에서 ARIA 속성, alt 텍스트 등 검사
- **순환 의존성**: `check_graph_for_cycles.js`로 반응형 선언 간 순환 감지

## Internal Flow

```
         Parse 단계의 AST.Root
                |
                v
    +-------------------------+
    |   create_scopes()       |
    |   스코프 트리 구축        |
    +-------------------------+
                |
                v
    +-------------------------+
    |   walk(ast, visitors)   |
    |   zimmerframe 순회       |
    |                         |
    |  +-------------------+  |
    |  | _ (와일드카드)     |  |  -- svelte-ignore 처리
    |  +-------------------+  |
    |  | Identifier        |  |  -- 변수 참조 해석
    |  | BindDirective     |  |  -- 바인딩 검증
    |  | Component         |  |  -- 컴포넌트 분석
    |  | RegularElement    |  |  -- 엘리먼트 + a11y 검사
    |  | EachBlock         |  |  -- 블록 스코프 분석
    |  | CallExpression    |  |  -- rune 호출 분석
    |  | ... (60+ visitors)|  |
    |  +-------------------+  |
    +-------------------------+
                |
                v
    +-------------------------+
    |   analyze_css()         |
    |   CSS 셀렉터 매칭 분석   |
    +-------------------------+
                |
                v
    +-------------------------+
    |   prune()               |
    |   미사용 CSS 제거 마킹   |
    +-------------------------+
                |
                v
    +-------------------------+
    |   warn_unused()         |
    |   미사용 CSS 경고        |
    +-------------------------+
                |
                v
      ComponentAnalysis 출력
      (enriched AST + 메타데이터)
```

## Dependencies

| 의존성 | 용도 |
|--------|------|
| `zimmerframe` | AST 트리 순회 (walk 함수) |
| `is-reference` | 식별자가 변수 참조인지 판별 |
| `acorn` | analyze 단계 내 추가 JS 파싱 (scope 생성 시) |

## Testing

```bash
# 컴파일러 에러 관련 테스트
pnpm test compiler-errors

# 유효성 검증 테스트
pnpm test validator

# CSS 관련 테스트
pnpm test css
```

## Related Notes

- [[Compiler-Parse]] - AST를 생성하는 이전 단계
- [[Compiler-Transform]] - 분석 결과를 기반으로 코드를 생성하는 다음 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인 개요
- [[System-Architecture]] - 시스템 아키텍처 전체 구조
