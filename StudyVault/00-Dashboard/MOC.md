---
module: dashboard
path: 00-Dashboard
keywords: MOC, onboarding, architecture, svelte
tags: [dashboard, onboarding]
---

# Svelte 5 StudyVault -- Map of Content

> [!info] Svelte 5 모노레포 온보딩 가이드
> 이 문서는 Svelte 5 (v5.55.0) 프레임워크 코드베이스에 새로 합류하는 개발자를 위한 중앙 허브입니다.
> 빠른 명령어와 파일 위치는 [[Quick-Reference]]를 참고하세요.

---

## Architecture Overview

Svelte 5는 **단일 패키지 모노레포** 구조를 채택하고 있습니다.

```
svelte-tutor/                (monorepo root)
├── packages/
│   └── svelte/              (핵심 패키지 -- 컴파일러 + 런타임)
│       └── src/
│           ├── compiler/    ← 3-phase 컴파일러
│           ├── internal/    ← 클라이언트 + 서버 런타임
│           ├── reactivity/  ← 시그널 기반 반응성
│           ├── store/       ← 스토어 API
│           └── ...
├── playgrounds/             (개발용 플레이그라운드)
└── sites/                   (공식 문서 사이트)
```

**핵심 분리 원칙**: 컴파일러(빌드 타임)와 런타임(실행 시간)이 명확히 분리되어 있습니다.

| 계층 | 역할 | 핵심 기술 |
|------|------|-----------|
| Compiler | `.svelte` 파일을 JavaScript로 변환 | acorn, esrap, zimmerframe, magic-string |
| Client Runtime | 브라우저에서 반응성 + DOM 관리 | Signal (Runes), Proxy, fine-grained reactivity |
| Server Runtime | SSR HTML 문자열 생성 | Lightweight renderer, no reactivity overhead |

---

## Module Map

| # | 모듈 | 목적 | 주요 진입점 | 노트 |
|---|------|------|-------------|------|
| 1 | **Parse** | 소스 코드를 AST로 변환 | `src/compiler/phases/1-parse/index.js` | [[Compiler-Parse]], [[Compilation-Pipeline]] |
| 2 | **Analyze** | AST 분석, 스코프 해석, 바인딩 | `src/compiler/phases/2-analyze/index.js` | [[Compiler-Analyze]], [[Compilation-Pipeline]] |
| 3 | **Transform** | AST를 JS/CSS 코드로 변환 | `src/compiler/phases/3-transform/index.js` | [[Compiler-Transform]], [[Compilation-Pipeline]] |
| 4 | **Reactivity** | 시그널 기반 반응성 시스템 (Runes) | `src/internal/client/reactivity/sources.js` | [[Reactivity-System]], [[System-Architecture]] |
| 5 | **Client Runtime** | DOM 렌더링, 이벤트, 라이프사이클 | `src/internal/client/index.js` | [[Client-Runtime]], [[DOM-Operations]], [[System-Architecture]] |
| 6 | **Server Runtime** | SSR 렌더러, HTML 문자열 생성 | `src/internal/server/index.js` | [[Server-Runtime]], [[System-Architecture]] |
| 7 | **Public API** | `svelte`, `svelte/compiler` 진입점 | `src/index-client.js`, `src/compiler/index.js` | [[Public-API]], [[Store-API]], [[Motion-API]], [[System-Architecture]] |
| 8 | **DevOps** | CI/CD, 빌드, 테스트 설정 | `.github/workflows/ci.yml` | [[Build-System]], [[Testing-Infrastructure]], [[CI-CD]], [[Quick-Reference]] |

> [!note] 경로 기준
> 모든 경로는 `packages/svelte/` 기준입니다.

---

## API Surface

### Compiler API (`svelte/compiler`) → [[Public-API]]

| 함수 | 설명 | 태그 |
|------|------|------|
| `compile(source, options)` | `.svelte` 소스를 JS/CSS로 컴파일 | #api-compiler |
| `compileModule(source, options)` | `.svelte.js` 모듈 파일 컴파일 | #api-compiler |
| `parse(source, options)` | 소스를 AST로 파싱만 수행 | #api-compiler |
| `parseCss(source)` | CSS를 AST로 파싱 | #api-compiler |
| `preprocess(source, preprocessors)` | 전처리기 실행 (scss, typescript 등) | #api-compiler |

### Runtime API (`svelte`) → [[Client-Runtime]], [[Server-Runtime]]

| 함수 | 설명 | 태그 |
|------|------|------|
| `mount(component, options)` | 컴포넌트를 DOM에 마운트 | #api-runtime |
| `hydrate(component, options)` | SSR HTML을 클라이언트에서 하이드레이션 | #api-runtime |
| `unmount(component)` | 컴포넌트를 DOM에서 제거 | #api-runtime |
| `flushSync()` | 대기 중인 반응성 업데이트를 즉시 실행 | #api-runtime |
| `tick()` | 다음 마이크로태스크까지 대기 | #api-runtime |

### Store API (`svelte/store`) → [[Store-API]]

| 함수 | 설명 | 태그 |
|------|------|------|
| `writable(value)` | 읽기/쓰기 가능한 스토어 생성 | #api-store |
| `readable(value, start)` | 읽기 전용 스토어 생성 | #api-store |
| `derived(stores, fn)` | 파생 스토어 생성 | #api-store |

---

## Getting Started

### Prerequisites

| 요구사항 | 최소 버전 |
|----------|-----------|
| Node.js | >= 18 |
| pnpm | >= 9 |

### 명령어

```bash
# 1. 의존성 설치
pnpm install

# 2. 빌드
pnpm build

# 3. 개발 모드 (파일 변경 시 자동 리빌드)
cd packages/svelte && pnpm dev

# 4. 테스트
pnpm test

# 5. 린트
pnpm lint
```

> [!tip] 빠른 시작
> 자세한 명령어와 환경 설정은 [[Quick-Reference]]를 참고하세요.

---

## Tag Index

| 태그 | 분류 | 관련 노트 |
|------|------|-----------|
| #arch-compiler | Architecture | [[System-Architecture]], [[Compilation-Pipeline]] |
| #arch-runtime | Architecture | [[System-Architecture]], [[Request-Flow]] |
| #arch-monorepo | Architecture | [[System-Architecture]] |
| #module-parse | Module | [[Compilation-Pipeline]] |
| #module-analyze | Module | [[Compilation-Pipeline]] |
| #module-transform | Module | [[Compilation-Pipeline]] |
| #module-reactivity | Module | [[System-Architecture]] |
| #module-client | Module | [[System-Architecture]], [[Request-Flow]] |
| #module-server | Module | [[System-Architecture]], [[Request-Flow]] |
| #module-store | Module | [[System-Architecture]] |
| #module-motion | Module | [[System-Architecture]] |
| #module-devops | Module | [[Quick-Reference]] |
| #pattern-visitor | Pattern | [[Compilation-Pipeline]] |
| #pattern-signal | Pattern | [[System-Architecture]] |
| #pattern-proxy | Pattern | [[System-Architecture]] |
| #pattern-dual-runtime | Pattern | [[System-Architecture]], [[Request-Flow]] |
| #api-compiler | API | [[Compilation-Pipeline]] |
| #api-runtime | API | [[Request-Flow]] |
| #api-store | API | [[System-Architecture]] |
| #config-build | Config | [[Quick-Reference]] |
| #config-test | Config | [[Quick-Reference]] |

---

## Onboarding Path

> [!success] 권장 학습 순서
> 아래 순서대로 읽으면 전체 아키텍처를 체계적으로 이해할 수 있습니다.

1. [[System-Architecture]] --- 전체 구조 파악
2. [[Compilation-Pipeline]] --- 컴파일러 3단계 이해
3. [[Request-Flow]] --- 요청 흐름 추적
4. [[Compiler-Parse]] --- 파싱 심화
5. [[Compiler-Analyze]] --- 분석 심화
6. [[Compiler-Transform]] --- 변환 심화
7. [[Reactivity-System]] --- 시그널/Runes 심화
8. [[Client-Runtime]] --- DOM 런타임 심화
9. [[DOM-Operations]] --- DOM 조작 심화
10. [[Server-Runtime]] --- SSR 심화
11. [[Public-API]] --- 외부 API 인터페이스
12. [[Store-API]] --- 스토어 API
13. [[Motion-API]] --- 모션/애니메이션 API
14. [[Build-System]] --- 빌드 시스템
15. [[Testing-Infrastructure]] --- 테스트 인프라
16. [[CI-CD]] --- CI/CD 설정
17. Exercises --- 실습 과제
    - [[Compiler-Exercises]] --- 컴파일러 실습
    - [[Runtime-Exercises]] --- 런타임 실습
    - [[Reactivity-Exercises]] --- 반응성 실습
    - [[Integration-Exercises]] --- 통합 실습

---

## Practice & Exercise Files

### Practice Files (실습)

| 모듈 | Practice 노트 |
|------|---------------|
| Compiler Parse | [[Practice-Parse]] |
| Compiler Analyze | [[Practice-Analyze]] |
| Compiler Transform | [[Practice-Transform]] |
| Reactivity | [[Practice-Reactivity]] |
| Client Runtime | [[Practice-Client-Runtime]] |
| Server Runtime | [[Practice-Server-Runtime]] |
| Public API | [[Practice-Public-API]] |

### Exercise Files (연습 과제)

| 분류 | Exercise 노트 |
|------|---------------|
| Compiler | [[Compiler-Exercises]] |
| Runtime | [[Runtime-Exercises]] |
| Reactivity | [[Reactivity-Exercises]] |
| Integration | [[Integration-Exercises]] |
