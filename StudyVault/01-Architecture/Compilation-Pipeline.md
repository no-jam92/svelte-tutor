---
module: architecture
path: 01-Architecture
keywords: compiler, pipeline, parse, analyze, transform
tags:
  - "#arch-compiler"
  - "#module-parse"
  - "#module-analyze"
  - "#module-transform"
  - "#pattern-visitor"
  - "#api-compiler"
---

# Compilation Pipeline

> [!abstract] 목적
> Svelte 5 컴파일러의 3단계 파이프라인(Parse → Analyze → Transform)을 상세히 설명합니다.
> 각 단계의 입력, 출력, 핵심 파일을 이해하는 것이 목표입니다.

---

## Pipeline Overview

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                    compile(source, options)                         │
 │                  packages/svelte/src/compiler/index.js              │
 └───────────────────────────┬─────────────────────────────────────────┘
                             |
                             v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  PHASE 1: PARSE                                                    │
 │  phases/1-parse/index.js                                           │
 │                                                                    │
 │  Input:  .svelte 소스 문자열                                          │
 │  Output: Svelte AST (Fragment, Script, Style 노드 포함)              │
 │                                                                    │
 │  +-----------+     +-------------------+     +-----------+         │
 │  | acorn     |     | Template Tokenizer|     | CSS       |         │
 │  | (JS 파싱) |      | (HTML-like 파싱)   |     | (스타일)    |         │
 │  +-----------+     +-------------------+     +-----------+         │
 │       |                     |                      |               │
 │       +---------------------+----------------------+               │
 │                             |                                      │
 │                        Svelte AST                                  │
 └───────────────────────────┬─────────────────────────────────────────┘
                             |
                             v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  PHASE 2: ANALYZE                                                  │
 │  phases/2-analyze/index.js                                         │
 │                                                                    │
 │  Input:  Svelte AST                                                │
 │  Output: Annotated AST (스코프, 바인딩, 의존성 정보 첨부)                 │
 │                                                                    │
 │  +------------------+  +------------------+  +-----------------+   │
 │  | Scope Resolution |  | Binding Analysis |  | Dependency Graph|   │
 │  | (변수 스코프)       |  | (바인딩 분석)       |  | (의존성 그래프)    |   │
 │  +------------------+  +------------------+  +-----------------+   │
 │           |                     |                     |            │
 │           +------  61 visitors (zimmerframe)  --------+            │
 │                             |                                      │
 │                    Annotated Svelte AST                             │
 └───────────────────────────┬─────────────────────────────────────────┘
                             |
                             v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  PHASE 3: TRANSFORM                                                │
 │  phases/3-transform/index.js                                       │
 │                                                                    │
 │  Input:  Annotated AST + options                                   │
 │  Output: { js: {code, map}, css: {code, map}, warnings, metadata } │
 │                                                                    │
 │  +-------------------+          +-------------------+              │
 │  | Client Codegen    |    OR    | Server Codegen    |              │
 │  | 3-transform/client|          | 3-transform/server|              │
 │  | (DOM 조작 코드)     |          | (HTML 문자열 코드)   |              │
 │  +-------------------+          +-------------------+              │
 │           |                              |                         │
 │           v                              v                         │
 │     esrap (AST → JS)             esrap (AST → JS)                  │
 │     + source maps                + source maps                     │
 └─────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Parse #module-parse

> [!info] 소스 문자열을 구조화된 AST로 변환하는 단계

### 진입점

`packages/svelte/src/compiler/phases/1-parse/index.js`

### 핵심 파일

| 파일/디렉토리 | 역할 |
|---------------|------|
| `1-parse/index.js` | 파싱 오케스트레이션 |
| `1-parse/acorn.js` | acorn 설정 및 JS 파싱 래퍼 |
| `1-parse/read/` | 토큰 읽기 유틸 (expression, script, style 등) |
| `1-parse/state/` | 상태 머신 기반 템플릿 토크나이저 |
| `1-parse/utils/` | 파싱 유틸리티 함수 |
| `1-parse/remove_typescript_nodes.js` | TypeScript 구문 제거 |

### 동작 방식

1. `<script>` 블록 → **acorn**으로 JavaScript AST 생성
2. 템플릿 영역 → **커스텀 상태 머신 토크나이저**로 HTML-like 구문 파싱
3. `<style>` 블록 → CSS 파서로 스타일 AST 생성
4. 세 결과를 합쳐 **Svelte AST** 구성

```
.svelte 파일 구조:

  <script>            ──→ acorn (JS parser)
    let count = 0;
  </script>

  <button>            ──→ Custom template tokenizer
    {count}                (state machine in state/)
  </button>

  <style>             ──→ CSS parser
    button { ... }
  </style>
```

---

## Phase 2: Analyze #module-analyze

> [!info] AST를 순회하며 의미 분석을 수행하는 단계

### 진입점

`packages/svelte/src/compiler/phases/2-analyze/index.js`

### 핵심 파일

| 파일/디렉토리 | 역할 |
|---------------|------|
| `2-analyze/index.js` | 분석 오케스트레이션 |
| `2-analyze/visitors/` | **61개의 AST visitor 파일** (노드 타입별 분석 로직) |
| `2-analyze/css/` | CSS 스코핑 분석 |
| `2-analyze/utils/` | 분석 유틸리티 |
| `compiler/scope.js` | 스코프 체인 관리 |
| `compiler/bindings.js` | 바인딩 분석 |

### 주요 분석 작업

| 작업 | 설명 |
|------|------|
| **Scope Resolution** | 변수 선언 → 참조 매핑, 렉시컬 스코프 체인 구축 |
| **Binding Analysis** | `$state`, `$derived`, `$props` 등 Rune 바인딩 식별 |
| **Dependency Graph** | 반응성 의존성 추적 (어떤 값이 어떤 값에 의존하는지) |
| **Validation** | 잘못된 구문, 비호환 패턴 검출 → `errors.js` / `warnings.js` |
| **CSS Scoping** | 컴포넌트 스코프 CSS 클래스 해시 계산 |

### Visitor Pattern

```
walk(ast, {
    // 각 노드 타입에 대한 visitor 함수
    ExpressionTag(node, context) { ... },
    Element(node, context) { ... },
    Component(node, context) { ... },
    BindDirective(node, context) { ... },
    // ... 61개 visitor
})
```

> [!tip] zimmerframe 라이브러리
> `zimmerframe`은 AST를 깊이 우선 순회하면서 enter/leave 콜백을 실행합니다.
> 각 visitor는 `context.visit(child)`로 자식 노드를 재귀 방문할 수 있습니다.

---

## Phase 3: Transform #module-transform

> [!info] 분석된 AST를 실행 가능한 JavaScript/CSS 코드로 변환하는 단계

### 진입점

`packages/svelte/src/compiler/phases/3-transform/index.js`

### 핵심 파일

| 파일/디렉토리 | 역할 |
|---------------|------|
| `3-transform/index.js` | 변환 오케스트레이션 (client/server 분기) |
| `3-transform/client/` | **클라이언트 코드 생성** (DOM 조작 코드) |
| `3-transform/server/` | **서버 코드 생성** (HTML 문자열 생성 코드) |
| `3-transform/shared/` | client/server 공통 변환 로직 |
| `3-transform/css/` | CSS 변환 (스코핑 적용) |
| `3-transform/utils.js` | 변환 유틸리티 |

### Client vs Server 코드 생성

| 구분 | Client (`generate: 'client'`) | Server (`generate: 'server'`) |
|------|-------------------------------|-------------------------------|
| 반응성 | `$.state()`, `$.derived()`, `$.effect()` 호출 | 없음 (정적 값) |
| DOM | `$.element()`, `$.text()` 등 fine-grained 조작 | HTML 문자열 연결 |
| 이벤트 | `$.event()` 이벤트 리스너 등록 | 없음 |
| 출력 크기 | 비교적 큼 (런타임 호출 포함) | 작음 (문자열 생성만) |

### 코드 생성 도구

```
  Annotated AST
       |
       v
  esrap (AST → JavaScript 소스 코드)
       |
       v
  @jridgewell/sourcemap-codec (소스 맵 생성)
       |
       v
  { code: "...", map: {...} }
```

---

## Input / Output

### Input

`.svelte` 파일 소스 문자열 + 컴파일러 옵션:

```javascript
import { compile } from 'svelte/compiler';

const result = compile(source, {
    generate: 'client',  // 또는 'server'
    dev: false,          // 개발 모드
    css: 'injected',     // CSS 처리 방식
    // ... 기타 옵션
});
```

### Output

```javascript
{
    js: {
        code: "...",      // 생성된 JavaScript 코드
        map: { ... }      // 소스 맵
    },
    css: {
        code: "...",      // 스코핑된 CSS 코드
        map: { ... }      // CSS 소스 맵
    },
    warnings: [ ... ],    // 컴파일 경고 배열
    metadata: {
        runes: true,      // Runes 모드 여부
        // ... 기타 메타데이터
    }
}
```

---

## Related Notes

- [[System-Architecture]] -- 전체 시스템 아키텍처
- [[Request-Flow]] -- 컴파일 → 렌더링 전체 흐름
- [[MOC]] -- 모듈 맵 및 학습 경로
- [[Quick-Reference]] -- 디버깅 진입점 파일 목록
