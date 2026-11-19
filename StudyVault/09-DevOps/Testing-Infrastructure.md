---
module: devops-testing
path: packages/svelte/tests
keywords: vitest, playwright, testing, test-suites
tags: [module-devops, config-test]
---

# Testing Infrastructure

> [!info] 학습 목표
> Svelte의 테스트 인프라와 테스트 스위트 구성을 이해한다.

**난이도:** ★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `vitest.config.js` | 루트 Vitest 설정 |
| `packages/svelte/tests/` | 모든 테스트 스위트 |
| `packages/svelte/tests/helpers.js` | 공통 테스트 헬퍼 |
| `packages/svelte/tests/html_equal.js` | HTML 비교 유틸 |
| `packages/svelte/tests/animation-helpers.js` | 애니메이션 테스트 헬퍼 |
| `packages/svelte/tests/suite.ts` | 테스트 스위트 러너 |

## Vitest 설정

`vitest.config.js`의 핵심 설정:

```
resolve.alias:
  svelte/* --> packages/svelte/src/* (브라우저 or 서버 export 자동 선택)

test:
  dir: '.'
  reporters: ['dot']
  include:
    - packages/svelte/**/*.test.ts
    - packages/svelte/tests/*/test.ts
    - packages/svelte/tests/runtime-browser/test-ssr.ts
  exclude:
    - **/samples/** (테스트 샘플 데이터 제외)
  coverage:
    - provider: v8, include: packages/svelte/src/**
```

> [!note] import 별칭 분기
> `importer`가 `_output/server`를 포함하면 서버 export를, 그렇지 않으면 브라우저 export를 사용한다. 이를 통해 같은 테스트 코드로 클라이언트/서버 양쪽을 테스트할 수 있다.

## 테스트 스위트 카테고리

| 디렉토리 | 설명 | 범주 |
|----------|------|------|
| `compiler-errors` | 컴파일러 에러 메시지 검증 | Compiler |
| `css` | CSS 스코핑, 최적화 테스트 | Compiler |
| `css-parse.test.ts` | CSS 파서 단위 테스트 | Compiler |
| `parser-legacy` | Svelte 4 레거시 파서 테스트 | Compiler |
| `parser-modern` | Svelte 5 모던 파서 테스트 | Compiler |
| `print` | AST -> 코드 출력 테스트 | Compiler |
| `validator` | 유효성 검사 (경고/에러) 테스트 | Compiler |
| `sourcemaps` | 소스맵 생성 정확성 테스트 | Compiler |
| `snapshot` | 컴파일 출력 스냅샷 테스트 | Compiler |
| `migrate` | Svelte 4 -> 5 마이그레이션 테스트 | Compiler |
| `preprocess` | 전처리기 테스트 | Compiler |
| `runtime-runes` | Svelte 5 Runes 런타임 테스트 | Runtime |
| `runtime-legacy` | Svelte 4 레거시 런타임 테스트 | Runtime |
| `runtime-production` | 프로덕션 모드 런타임 테스트 | Runtime |
| `runtime-browser` | 브라우저 환경 런타임 테스트 (Playwright) | Runtime |
| `runtime-xhtml` | XHTML 모드 런타임 테스트 | Runtime |
| `hydration` | SSR 하이드레이션 테스트 | Runtime |
| `server-side-rendering` | SSR 출력 검증 테스트 | Runtime |
| `signals` | 반응성 시그널 단위 테스트 | Reactivity |
| `store` | Store API 테스트 | Store |
| `motion` | Spring/Tweened 애니메이션 테스트 | Motion |
| `types` | TypeScript 타입 검증 테스트 | Types |
| `manual` | 수동 테스트 케이스 | Manual |

## 테스트 헬퍼

### helpers.js

테스트에서 공통으로 사용하는 유틸리티:
- 컴포넌트 컴파일 및 마운트
- DOM 상태 검증
- 이벤트 시뮬레이션

### html_equal.js

HTML 문자열 비교를 위한 유틸. 공백, 속성 순서 등의 차이를 무시하고 의미적으로 동등한지 비교한다.

### animation-helpers.js

rAF(requestAnimationFrame) 모킹 및 애니메이션 타이밍 제어를 위한 헬퍼.

### suite.ts

테스트 스위트의 공통 러너. `samples/` 디렉토리의 각 테스트 케이스를 자동으로 발견하고 실행한다.

## Playwright 브라우저 테스트

`runtime-browser` 테스트는 실제 브라우저 환경이 필요하다:

```
CI 환경:
  pnpm playwright install chromium  (Chromium만 설치)
  PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1  (자동 다운로드 비활성화)
```

## 테스트 실행 방법

### 전체 테스트

```bash
pnpm test
```

### 특정 스위트 실행

```bash
# 특정 테스트 디렉토리
pnpm test runtime-runes
pnpm test signals
pnpm test compiler-errors

# 특정 테스트 파일
pnpm vitest packages/svelte/tests/store/test.ts

# 패턴 매칭
pnpm vitest --reporter=verbose -t "spring"
```

### 커버리지 리포트

```bash
pnpm vitest --coverage
# 결과: coverage/ 디렉토리에 lcov + html 리포트
```

### 비동기 비활성화 테스트

```bash
SVELTE_NO_ASYNC=true pnpm test runtime-runes
```

## 새 테스트 추가 방법

### 샘플 기반 테스트 (대부분의 스위트)

```
tests/<suite>/samples/<test-name>/
  +-- _config.js      테스트 설정 (props, expected 등)
  +-- main.svelte     테스트 컴포넌트
  +-- _expected/      기대 출력 (선택)
```

### 단위 테스트

```
tests/<suite>/<name>.test.ts
```

> [!tip] 테스트 디버깅
> `test.only` 또는 `it.only`로 특정 테스트만 실행할 수 있다. `--reporter=verbose`로 상세 출력을 볼 수 있다.

## Related Notes

- [[CI-CD]] --- CI에서 테스트 실행 방식
- [[Build-System]] --- 빌드 시스템과 테스트의 관계
- [[Integration-Exercises]] --- 통합 연습문제
