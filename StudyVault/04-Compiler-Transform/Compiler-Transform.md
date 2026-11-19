---
module: compiler-transform
path: packages/svelte/src/compiler/phases/3-transform
keywords: transform, codegen, client, server, esrap, sourcemap, CSS
tags: [module-transform, arch-compiler]
---

# Compiler Transform (3-transform)

> **Purpose:** 분석된 AST를 실행 가능한 JavaScript로 변환하는 단계
> **Importance:** ★★★

## Overview

Transform 단계는 컴파일러 파이프라인의 마지막 단계입니다. [[Compiler-Analyze]] 단계에서 생성된 `ComponentAnalysis` 객체를 입력받아, **클라이언트용** 또는 **서버용** JavaScript 코드와 스코프된 CSS를 출력합니다. 코드 생성에는 **esrap** 라이브러리를 사용하며, 소스맵도 함께 생성됩니다.

## Key Files

### 최상위 파일

| 파일 | 역할 |
|------|------|
| `index.js` | `transform_component()`, `transform_module()` 내보내기 (진입점) |
| `utils.js` | 변환 공통 유틸리티 |
| `types.d.ts` | 변환 관련 타입 정의 |

### Client 변환 (client/)

| 파일 | 역할 |
|------|------|
| `transform-client.js` | `client_component()`, `client_module()` - 클라이언트 변환 진입점 |
| `utils.js` | 클라이언트 변환 유틸리티 |
| `types.d.ts` | 클라이언트 변환 타입 |
| `transform-template/index.js` | 템플릿 -> DOM 조작 코드 변환 |
| `transform-template/template.js` | 템플릿 문자열 생성 |
| `transform-template/fix-attribute-casing.js` | SVG/MathML 속성 대소문자 보정 |

**Client Visitors** (`client/visitors/`) - 50개 이상:

| 카테고리 | 주요 Visitor |
|----------|-------------|
| **블록** | `IfBlock.js`, `EachBlock.js`, `AwaitBlock.js`, `KeyBlock.js`, `SnippetBlock.js` |
| **컴포넌트** | `Component.js`, `SvelteComponent.js`, `SvelteElement.js`, `SvelteSelf.js` |
| **엘리먼트** | `RegularElement.js`, `SlotElement.js`, `TitleElement.js`, `Fragment.js` |
| **디렉티브** | `BindDirective.js`, `OnDirective.js`, `UseDirective.js`, `TransitionDirective.js`, `AnimateDirective.js`, `LetDirective.js` |
| **표현식** | `Identifier.js`, `MemberExpression.js`, `CallExpression.js`, `AssignmentExpression.js`, `UpdateExpression.js`, `AwaitExpression.js`, `BinaryExpression.js` |
| **문장** | `ExpressionStatement.js`, `VariableDeclaration.js`, `LabeledStatement.js`, `BlockStatement.js`, `BreakStatement.js`, `ForOfStatement.js` |
| **기타** | `Program.js`, `Comment.js`, `HtmlTag.js`, `ConstTag.js`, `DebugTag.js`, `RenderTag.js`, `AttachTag.js`, `SpreadAttribute.js`, `ExportNamedDeclaration.js`, `FunctionDeclaration.js`, `FunctionExpression.js`, `ArrowFunctionExpression.js` |
| **특수 엘리먼트** | `SvelteBody.js`, `SvelteWindow.js`, `SvelteDocument.js`, `SvelteHead.js`, `SvelteBoundary.js`, `SvelteFragment.js` |

### Server 변환 (server/)

| 파일 | 역할 |
|------|------|
| `transform-server.js` | `server_component()`, `server_module()` - SSR 변환 진입점 |
| `types.d.ts` | 서버 변환 타입 |

**Server Visitors** (`server/visitors/`) - 30개 이상:

| 카테고리 | 주요 Visitor |
|----------|-------------|
| **블록** | `IfBlock.js`, `EachBlock.js`, `AwaitBlock.js`, `KeyBlock.js`, `SnippetBlock.js` |
| **컴포넌트** | `Component.js`, `SvelteComponent.js`, `SvelteElement.js`, `SvelteSelf.js` |
| **엘리먼트** | `RegularElement.js`, `SlotElement.js`, `TitleElement.js`, `Fragment.js` |
| **표현식** | `Identifier.js`, `MemberExpression.js`, `CallExpression.js`, `AssignmentExpression.js`, `UpdateExpression.js`, `AwaitExpression.js` |
| **기타** | `Program.js`, `HtmlTag.js`, `ConstTag.js`, `DebugTag.js`, `RenderTag.js`, `SpreadAttribute.js`, `VariableDeclaration.js`, `LabeledStatement.js`, `ExpressionStatement.js`, `PropertyDefinition.js`, `ClassBody.js` |

### Shared (shared/)

| 파일 | 역할 |
|------|------|
| `assignments.js` | 할당문 변환 공통 로직 |
| `transform-async.js` | 비동기 변환 처리 |

### CSS 변환 (css/)

| 파일 | 역할 |
|------|------|
| `index.js` | `render_stylesheet()` - CSS 출력 생성 및 스코핑 |

## Client vs Server Transform

```
                  ComponentAnalysis
                        |
          +-------------+-------------+
          |                           |
          v                           v
  +------------------+      +------------------+
  | Client Transform |      | Server Transform |
  | (generate='client')|    | (generate='server')|
  +------------------+      +------------------+
          |                           |
          v                           v
  +------------------+      +------------------+
  | DOM 조작 코드     |      | 문자열 연결 코드  |
  | - $.element()    |      | - $$payload 구성  |
  | - $.text()       |      | - HTML 문자열 생성 |
  | - $.effect()     |      | - 서버 전용 최적화 |
  | - $.bind()       |      |                  |
  | - 이벤트 핸들러   |      |                  |
  +------------------+      +------------------+
```

### Client Transform

- **DOM 기반 렌더링**: `document.createElement()` 등을 사용하는 런타임 코드 생성
- **반응성**: `$state`, `$derived` -> `$.state()`, `$.derived()` 런타임 호출로 변환
- **이펙트**: `$effect` -> `$.effect()` 래퍼로 변환
- **템플릿 최적화**: `transform-template/`에서 정적 HTML을 템플릿 문자열로 최적화
- **이벤트**: `on:click` -> `$.event("click", ...)` 로 변환

### Server Transform

- **문자열 기반 렌더링**: HTML 문자열을 조합하여 SSR 출력 생성
- **`$$payload`**: 서버 렌더링 결과를 담는 payload 객체
- **경량화**: 반응성, 이벤트 핸들러, 트랜지션 등 클라이언트 전용 기능 생략
- **동일한 visitor 패턴**을 사용하지만 출력이 다름

## Code Generation with esrap

**esrap**은 ESTree AST를 JavaScript 소스 코드로 변환하는 프린터입니다:

```javascript
import { print } from 'esrap';
import ts from 'esrap/languages/ts';

const js = print(
  program,                    // ESTree AST (변환 결과)
  ts({ comments: analysis.comments }),  // TypeScript 언어 플러그인
  {
    sourceMapContent: source,     // 원본 소스 (소스맵용)
    sourceMapSource: js_source_name  // 소스맵 파일명
  }
);
```

반환값:
- `js.code` - 생성된 JavaScript 코드 문자열
- `js.map` - 소스맵 객체

## Source Map Generation

소스맵은 두 단계로 생성됩니다:

1. **esrap의 `print()`** 에서 AST 노드의 `start`/`end` 위치를 기반으로 기본 소스맵 생성
2. **`merge_with_preprocessor_map()`** 에서 전처리기(preprocessor)의 소스맵과 병합
   - `svelte-preprocess` 등으로 TypeScript, SCSS를 변환한 경우 해당 소스맵과 결합

## CSS Transformation

`render_stylesheet()` (`css/index.js`)에서 처리:

- **스코핑**: 각 셀렉터에 컴포넌트 고유 해시를 추가 (`.foo` -> `.foo.svelte-abc123`)
- **미사용 제거**: analyze 단계에서 `prune()`으로 마킹된 미사용 규칙 제거
- **소스맵**: CSS용 소스맵도 별도 생성

## Internal Flow

```
    ComponentAnalysis + source + options
                    |
                    v
        +------------------------+
        | options.generate 확인   |
        +---+----------------+---+
            |                |
     'client'|          'server'|
            v                v
  +------------------+ +------------------+
  |client_component()| |server_component()|
  |   visitor 순회    | |   visitor 순회    |
  |   ESTree AST 생성 | |   ESTree AST 생성 |
  +------------------+ +------------------+
            |                |
            +-------+--------+
                    |
                    v
          +-------------------+
          | esrap print()     |
          | AST -> JS 코드    |
          | + 소스맵 생성      |
          +-------------------+
                    |
                    v
    +-------------------------------+
    | merge_with_preprocessor_map() |
    | 전처리기 소스맵과 병합          |
    +-------------------------------+
                    |
                    v
          +-------------------+
          | render_stylesheet()|
          | CSS 스코핑 + 출력  |
          | (CSS가 있는 경우)   |
          +-------------------+
                    |
                    v
            CompileResult
        { js, css, warnings,
          metadata, ast }
```

## Dependencies

| 의존성 | 용도 |
|--------|------|
| `esrap` | ESTree AST -> JavaScript 코드 생성 + 소스맵 |
| `esrap/languages/ts` | TypeScript 언어 지원 플러그인 |
| `zimmerframe` | AST 순회 (visitor 실행) |

## 설정

Transform 단계의 동작을 결정하는 주요 컴파일러 옵션:

| 옵션 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `generate` | `'client' \| 'server'` | `'client'` | 출력 대상. `'client'`는 DOM 조작 코드, `'server'`는 문자열 렌더링 코드 생성 |
| `dev` | `boolean` | `false` | 개발 모드. `true`면 디버깅 코드, 소유권 검증, `$inspect` 지원 코드 포함 |
| `css` | `'injected' \| 'external' \| 'none'` | `'injected'` | CSS 출력 방식. `'injected'`는 JS에 인라인, `'external'`은 별도 파일로 출력 |
| `hmr` | `boolean` | `false` | Hot Module Replacement 지원 코드 생성 여부 |
| `modernAst` | `boolean` | `false` | 모던 AST 형식으로 메타데이터 출력 |
| `customElement` | `boolean` | `false` | 커스텀 엘리먼트용 래퍼 코드 생성 |
| `sourcemap` | `object \| string` | — | 전처리기 소스맵. `merge_with_preprocessor_map()`에서 병합 |
| `discloseVersion` | `boolean` | `true` | 생성 코드에 Svelte 버전 정보 포함 여부 |
| `compatibility.componentApi` | `4 \| 5` | `5` | Svelte 4 호환 컴포넌트 API 래퍼 생성 |

> [!tip] `generate` 옵션의 영향
> `generate: 'client'`는 `client_component()`를, `generate: 'server'`는 `server_component()`를 호출한다. 각각 완전히 다른 visitor 세트를 사용하여 다른 출력을 생성한다.

## Testing

```bash
# 전체 컴파일러 테스트
pnpm test compiler

# 스냅샷 테스트 (생성된 코드 비교)
pnpm test snapshot

# SSR 테스트
pnpm test ssr

# CSS 변환 테스트
pnpm test css
```

> [!tip] 디버깅 팁
> 생성된 코드를 확인하려면 `CompileResult.js.code`를 출력하세요. 소스맵은 `CompileResult.js.map`에 있습니다. REPL(https://svelte.dev/playground)에서도 변환 결과를 실시간으로 확인할 수 있습니다.

## Related Notes

- [[Compiler-Analyze]] - 분석 결과를 입력으로 받는 이전 단계
- [[Compiler-Parse]] - 파싱 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인 개요
- [[System-Architecture]] - 시스템 아키텍처 전체 구조
