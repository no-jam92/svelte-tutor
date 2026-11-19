---
module: architecture
path: 01-Architecture
keywords: architecture, monorepo, compiler, runtime, svelte
tags:
  - "#arch-compiler"
  - "#arch-runtime"
  - "#arch-monorepo"
  - "#pattern-signal"
  - "#pattern-proxy"
  - "#pattern-dual-runtime"
---

# System Architecture

> [!abstract] 목적
> Svelte 5 전체 시스템 아키텍처를 설명합니다.
> 컴파일러, 런타임, 반응성 시스템의 관계를 이해하는 것이 목표입니다.

---

## High-Level Architecture

```
                         Svelte 5 Architecture
 ================================================================

  .svelte source
       |
       v
 +------------------------------------------------------------------+
 |                      COMPILER (Build Time)                       |
 |                                                                  |
 |  +-------------+    +--------------+    +----------------+       |
 |  |   Phase 1   |    |   Phase 2    |    |    Phase 3     |       |
 |  |   PARSE     |--->|   ANALYZE    |--->|   TRANSFORM    |       |
 |  |             |    |              |    |                |       |
 |  | acorn +     |    | 60+ visitors |    | esrap codegen  |       |
 |  | template    |    | scope/bind   |    | source maps    |       |
 |  | tokenizer   |    | dep graph    |    | client/server  |       |
 |  +-------------+    +--------------+    +-------+--------+       |
 |                                                 |                |
 +------------------------------------------------------------------+
                                                   |
                              +--------------------+--------------------+
                              |                                         |
                              v                                         v
                 +------------------------+              +------------------------+
                 |   CLIENT RUNTIME       |              |   SERVER RUNTIME       |
                 |   (Browser)            |              |   (Node.js/Edge)       |
                 |                        |              |                        |
                 | +------------------+   |              | +------------------+   |
                 | | Reactivity       |   |              | | Renderer         |   |
                 | | (Signals/Runes)  |   |              | | (HTML string)    |   |
                 | |  $state          |   |              | +------------------+   |
                 | |  $derived        |   |              |                        |
                 | |  $effect         |   |              | - No reactivity        |
                 | +------------------+   |              | - No DOM               |
                 | +------------------+   |              | - Lightweight          |
                 | | DOM Operations   |   |              +------------------------+
                 | | (fine-grained)   |   |
                 | +------------------+   |
                 +------------------------+
```

---

## Monorepo Structure

Svelte 5는 pnpm 기반 모노레포이지만, 핵심은 **단일 패키지** `packages/svelte`에 집중되어 있습니다.

```
svelte-tutor/
├── packages/
│   └── svelte/                    # 유일한 핵심 패키지
│       ├── src/
│       │   ├── compiler/          # 컴파일러 전체
│       │   │   ├── phases/
│       │   │   │   ├── 1-parse/   # Phase 1: 파싱
│       │   │   │   ├── 2-analyze/ # Phase 2: 분석
│       │   │   │   └── 3-transform/ # Phase 3: 변환
│       │   │   ├── errors.js      # 컴파일 에러 정의
│       │   │   ├── warnings.js    # 컴파일 경고 정의
│       │   │   └── index.js       # 컴파일러 진입점
│       │   ├── internal/
│       │   │   ├── client/        # 클라이언트 런타임
│       │   │   │   ├── reactivity/# 시그널 시스템
│       │   │   │   ├── dom/       # DOM 조작
│       │   │   │   ├── dev/       # 개발 모드 유틸
│       │   │   │   └── proxy.js   # 상태 프록시
│       │   │   └── server/        # 서버 런타임
│       │   │       ├── renderer.js# SSR 렌더러
│       │   │       └── index.js   # 서버 진입점
│       │   ├── reactivity/        # 공개 반응성 API
│       │   ├── store/             # 스토어 API (레거시 호환)
│       │   ├── motion/            # 트랜지션/애니메이션
│       │   ├── index-client.js    # 클라이언트 public API
│       │   └── index-server.js    # 서버 public API
│       └── tests/                 # 테스트 스위트
├── playgrounds/                   # 개발 플레이그라운드
├── sites/                         # 문서 사이트
└── .github/workflows/             # CI/CD 워크플로우
```

---

## Key Architectural Patterns

### 1. Visitor Pattern (AST 순회) #pattern-visitor

컴파일러는 AST를 순회하며 분석과 변환을 수행합니다. `zimmerframe` 라이브러리를 사용하여 각 노드 타입에 대한 visitor 함수를 정의합니다.

```
AST Tree
  |
  v
walk(ast, visitors)  ← zimmerframe
  |
  +---> ExpressionTag visitor
  +---> Element visitor
  +---> Component visitor
  +---> ... (60+ visitors in Phase 2)
```

> [!info] Phase 2 (Analyze)에는 약 61개의 visitor 파일이 존재합니다.

### 2. Signal-based Reactivity (시그널 반응성) #pattern-signal

Svelte 5의 Runes 시스템은 시그널 패턴을 사용합니다.

| Rune | 내부 구현 | 파일 |
|------|-----------|------|
| `$state` | Source signal (Proxy 래핑) | `reactivity/sources.js` + `proxy.js` |
| `$derived` | Derived signal (lazy evaluation) | `reactivity/deriveds.js` |
| `$effect` | Effect (auto-tracking subscriber) | `reactivity/effects.js` |

```
  $state(value)         $derived(fn)          $effect(fn)
       |                     |                      |
       v                     v                      v
  [Source Signal]  <--- [Derived Signal]  <--- [Effect]
       |                     |                      |
       +--- 값 변경 시 ------+---- 의존성 추적 ------+
                             |
                             v
                      DOM 업데이트 (fine-grained)
```

### 3. Proxy for State Tracking (프록시 상태 추적) #pattern-proxy

`$state`로 선언된 객체와 배열은 JavaScript `Proxy`로 래핑되어 프로퍼티 접근/수정을 자동 추적합니다.

- 파일: `packages/svelte/src/internal/client/proxy.js`
- 읽기 시 의존성 등록 (track)
- 쓰기 시 변경 알림 (notify)

### 4. Dual Runtime (이중 런타임) #pattern-dual-runtime

컴파일러는 **같은 `.svelte` 소스**에서 두 가지 다른 출력을 생성합니다:

| 대상 | 출력 | 런타임 | 특징 |
|------|------|--------|------|
| Client | `generate: 'client'` | `internal/client/` | 반응성, DOM 조작, 이벤트 |
| Server | `generate: 'server'` | `internal/server/` | HTML 문자열 생성, 경량 |

---

## Dependencies

### 핵심 외부 의존성

| 패키지 | 용도 | 사용 위치 |
|--------|------|-----------|
| **acorn** | JavaScript 파서 (ECMAScript 호환) | Phase 1 -- `<script>` 태그 내 JS 파싱 |
| **esrap** | 코드 생성기 (AST → JS 코드) | Phase 3 -- 변환된 AST를 코드로 출력 |
| **zimmerframe** | AST walker/visitor 프레임워크 | Phase 2/3 -- AST 순회 |
| **magic-string** | 문자열 변환 + source map 생성 | CSS 처리, 소스 맵 생성 |
| **@jridgewell/sourcemap-codec** | Source map 인코딩/디코딩 | Phase 3 -- 소스 맵 처리 |

### 내부 모듈 의존성

```
  compiler/index.js
       |
       +---> phases/1-parse   (acorn, custom tokenizer)
       +---> phases/2-analyze  (zimmerframe, scope.js, bindings.js)
       +---> phases/3-transform (esrap, magic-string)
                |
                +---> 3-transform/client/  (client codegen)
                +---> 3-transform/server/  (server codegen)

  internal/client/index.js
       |
       +---> reactivity/sources.js   ($state)
       +---> reactivity/deriveds.js  ($derived)
       +---> reactivity/effects.js   ($effect)
       +---> proxy.js                (Proxy wrapper)
       +---> dom/                    (DOM operations)
       +---> render.js               (component rendering)

  internal/server/index.js
       |
       +---> renderer.js             (SSR core)
       +---> hydration.js            (hydration markers)
```

---

## Related Notes

- [[MOC]] -- 전체 학습 경로 및 모듈 맵
- [[Compilation-Pipeline]] -- 컴파일러 3단계 파이프라인 상세
- [[Request-Flow]] -- 요청 흐름 (CSR + SSR) 추적
- [[Quick-Reference]] -- 명령어 및 파일 위치 빠른 참조
