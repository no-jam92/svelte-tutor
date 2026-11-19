---
module: exercises
path: 10-Exercises
keywords: practice, onboarding, integration, end-to-end, full-stack
tags: [arch-compiler, arch-runtime, practice]
---

# 통합 --- 온보딩 연습문제

> [!info] 연습 가이드
> 아래 문제들은 여러 모듈을 아우르는 통합적 이해를 요구합니다. 컴파일러, 런타임, 반응성, DevOps 등 다양한 영역의 지식을 종합하여 답하세요.

## 연습 1 [trace]: .svelte 파일에서 브라우저 렌더링까지

`.svelte` 파일이 브라우저에서 렌더링되기까지의 **전체 여정**을 추적하세요.

```svelte
<!-- App.svelte -->
<script>
  let count = $state(0);
</script>
<button onclick={() => count++}>{count}</button>
```

> [!answer]- 정답 보기
> ```
> App.svelte (소스)
>     |
>     v
> [1. Vite/Rollup 빌드 도구가 .svelte 파일 감지]
>     |
>     v
> [2. svelte/compiler 호출]
>     |
>     +---> Parse: 소스 --> Svelte AST
>     |     - <script> 블록 파싱 (acorn)
>     |     - 템플릿 파싱 (자체 파서)
>     |     - $state를 rune으로 인식
>     |
>     +---> Analyze: AST 분석
>     |     - count를 반응성 상태로 표시
>     |     - {count} 사용처의 의존성 추적
>     |     - onclick 핸들러 분석
>     |
>     +---> Transform: AST --> JavaScript
>           - $state(0) --> $.source(0)
>           - {count} --> $.get(count)
>           - 템플릿 --> DOM 생성 함수
>           |
>           v
> [3. 생성된 JavaScript 모듈]
>     |
>     v
> [4. 번들러가 의존성 해결 + 번들링]
>     |
>     v
> [5. 브라우저 로드]
>     |
>     v
> [6. mount(App, { target }) 호출]
>     |
>     +---> 반응성 루트 생성 (effect_root)
>     +---> 컴포넌트 함수 실행
>     |     - $.source(0) --> Signal 객체 생성
>     |     - DOM 노드 생성 (<button>)
>     |     - 텍스트 노드를 Signal에 바인딩
>     |     - onclick 이벤트 리스너 등록
>     +---> DOM을 target에 삽입
>     +---> onMount / $effect 실행
>     |
>     v
> [7. 인터랙티브 앱 완성]
>     |
>     v
> [8. 사용자가 버튼 클릭]
>     +---> $.set(count, $.get(count) + 1)
>     +---> 의존 effect들 dirty 표시
>     +---> 마이크로태스크에서 flush
>     +---> DOM 업데이트 (텍스트: "0" --> "1")
> ```

## 연습 2 [trace]: SSR에서 하이드레이션까지

서버에서 렌더링된 HTML이 클라이언트에서 hydrate되는 전체 과정을 추적하세요.

> [!answer]- 정답 보기
> **서버 측:**
> ```
> [1. 서버 요청 수신]
>     |
>     v
> [2. 컴파일러 (generate: 'server')]
>     - .svelte --> 서버용 JavaScript
>     - DOM 조작 대신 문자열 연결로 HTML 생성
>     |
>     v
> [3. render(App, { props })]
>     - src/internal/server/index.js 실행
>     - 컴포넌트 함수 실행 (동기)
>     - onMount는 noop (실행 안 함)
>     - onDestroy는 동기 실행
>     - HTML 문자열 + 하이드레이션 마커 생성
>     |
>     v
> [4. HTML 응답 전송]
>     - <button>0</button> + 하이드레이션 주석/속성
> ```
>
> **클라이언트 측:**
> ```
> [5. 브라우저가 HTML 수신 및 렌더링]
>     - 서버 HTML이 즉시 표시됨 (빠른 FCP)
>     |
>     v
> [6. JavaScript 번들 로드]
>     |
>     v
> [7. hydrate(App, { target })]
>     - mount()와 유사하지만 DOM을 새로 만들지 않음
>     - 기존 DOM 노드를 순회하며:
>       a. 하이드레이션 마커로 동적 부분 식별
>       b. 기존 텍스트/요소 노드를 Signal에 연결
>       c. 이벤트 리스너 부착
>       d. 반응성 시스템 활성화
>     |
>     v
> [8. 인터랙티브 앱 완성]
>     - 서버 HTML이 그대로 유지된 상태에서
>     - 반응성과 이벤트가 활성화됨
> ```
>
> **핵심 포인트:**
> - 서버/클라이언트 컴파일 결과가 다름 (`generate: 'server'` vs `'client'`)
> - 하이드레이션은 DOM을 재생성하지 않고 **재사용**
> - 불일치(mismatch)가 있으면 DEV 모드에서 경고

## 연습 3 [debug]: 개발 모드 전용 경고 디버깅

Svelte 앱이 프로덕션에서는 정상이지만 개발 모드에서 경고가 나올 때 어디를 확인해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **DEV 모드 경고의 출처:**
>
> 1. **DEV 전용 코드 확인:**
>    - Svelte 전체에서 `if (DEV)` 블록으로 감싸진 코드를 검색
>    - `import { DEV } from 'esm-env'`를 사용
>    - 프로덕션 빌드에서는 번들러가 `DEV = false`로 tree-shake
>
> 2. **주요 DEV 전용 파일들:**
>    - `src/internal/client/dev/` --- 개발 모드 전용 유틸리티
>    - `src/internal/client/errors.js` --- 에러 메시지 정의
>    - `src/internal/client/warnings.js` --- 경고 메시지 정의
>
> 3. **경고 유형별 확인 위치:**
>
>    | 경고 유형 | 확인 위치 |
>    |----------|-----------|
>    | 하이드레이션 불일치 | `src/internal/client/dom/hydration.js` |
>    | 접근성 (a11y) | `src/compiler/phases/2-analyze/` |
>    | 미사용 CSS | `src/compiler/phases/2-analyze/css/` |
>    | Rune 잘못된 사용 | `src/internal/client/errors.js` |
>    | 레거시 API 사용 | `src/index-client.js` |
>
> 4. **디버깅 순서:**
>    - 콘솔 경고 메시지 텍스트를 소스 코드에서 검색
>    - 해당 경고의 조건문 확인
>    - `if (DEV)` 블록 내부인지 확인
>    - 경고를 발생시키는 조건을 수정

## 연습 4 [config]: 새로운 export 추가

새로운 export를 svelte 패키지에 추가하려면 (예: `svelte/utils`) 어떤 파일들을 수정해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **수정해야 할 파일들:**
>
> **1. 소스 파일 생성**
> ```
> src/utils/index.js          --- 메인 구현
> src/utils/index-client.js   --- 클라이언트 전용 (필요시)
> src/utils/index-server.js   --- 서버 전용 (필요시)
> ```
>
> **2. package.json exports 맵 추가**
> ```json
> {
>   "exports": {
>     "./utils": {
>       "types": "./types/index.d.ts",
>       "browser": "./src/utils/index-client.js",
>       "default": "./src/utils/index-server.js"
>     }
>   }
> }
> ```
>
> **3. 타입 정의**
> ```
> src/utils/public.d.ts       --- 공개 타입 정의
> ```
> 그리고 `pnpm generate:types`로 타입 재생성
>
> **4. vitest.config.js 확인**
> - resolve alias에서 `svelte/*` 패턴이 이미 모든 하위 경로를 처리하므로
> - 새 export가 `package.json` exports에 있으면 테스트에서 자동 해결됨
>
> **5. Rollup 설정** (필요시)
> - 런타임 코드는 ES 모듈이므로 일반적으로 rollup 수정 불필요
> - UMD 번들이 필요한 경우에만 `rollup.config.js` 수정
>
> **6. 테스트 추가**
> ```
> tests/utils/test.ts         --- 단위 테스트
> tests/utils/samples/        --- 샘플 기반 테스트 (필요시)
> ```
>
> **7. CI 확인**
> - `pnpm build` 후 `git status --porcelain`에서 타입 변경이 감지되므로
> - 반드시 생성된 타입을 커밋에 포함

## 연습 5 [extend]: 새로운 내장 컴포넌트 추가

Svelte에 새로운 내장 컴포넌트 `<svelte:portal>`을 추가하는 전체 과정을 설명하세요. (DOM의 다른 위치에 자식을 렌더링하는 컴포넌트)

> [!answer]- 정답 보기
> **1단계: 파서** (`src/compiler/phases/1-parse/`)
> - `<svelte:portal>`을 특수 엘리먼트로 인식
> - `SveltePortal` AST 노드 타입 정의
> - 허용되는 속성 (예: `target`) 정의
> - 자식 노드 파싱
>
> **2단계: 분석기** (`src/compiler/phases/2-analyze/`)
> - `SveltePortal` 노드에 대한 visitor 추가
> - `target` 속성의 유효성 검사
> - 자식 노드의 반응성/스코프 분석
> - 잘못된 사용에 대한 경고/에러 추가
>
> **3단계: 변환기** (`src/compiler/phases/3-transform/`)
>
> 클라이언트 변환 (`client/`):
> ```js
> // 생성할 코드:
> $.portal(target_element, () => {
>   // 자식 노드 렌더링 코드
> });
> ```
>
> 서버 변환 (`server/`):
> - SSR에서의 처리 방법 결정
> - portal 내용을 별도 마커와 함께 출력
>
> **4단계: 클라이언트 런타임** (`src/internal/client/`)
> ```js
> // src/internal/client/dom/blocks/portal.js
> export function portal(target, fn) {
>   // target 요소 참조 획득
>   // fn()을 target 내부에 렌더링
>   // 원래 위치에는 앵커 노드만 남김
>   // 소멸 시 target에서 자식 제거
> }
> ```
>
> **5단계: 서버 런타임** (`src/internal/server/`)
> - SSR에서 portal 내용 처리 로직
>
> **6단계: 하이드레이션 지원**
> - 서버에서 렌더링된 portal을 클라이언트에서 hydrate하는 로직
>
> **7단계: 타입 정의**
> - `svelte/elements`에 `SveltePortalAttributes` 추가
> - `pnpm generate:types`
>
> **8단계: 테스트**
> - `tests/parser-modern/samples/svelte-portal/`
> - `tests/runtime-runes/samples/svelte-portal-*/`
> - `tests/hydration/samples/svelte-portal/`
> - `tests/server-side-rendering/samples/svelte-portal/`
> - `tests/snapshot/samples/svelte-portal/`
>
> **9단계: 문서 및 Changeset**
> - `pnpm changeset` (minor 변경)
> - 문서 사이트 업데이트

## 연습 6 [debug]: 빌드에서 런타임까지 에러 추적

사용자가 `import { something } from 'svelte'`를 했는데 `something is not exported from 'svelte'` 에러가 발생한다. 이 문제를 해결하려면 어떤 파일들을 확인해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **확인 순서:**
>
> 1. **package.json exports 맵:**
>    - `packages/svelte/package.json`의 `exports` 필드에서 `"."` 엔트리 확인
>    - 클라이언트(`browser`)와 서버(`default`) export 파일이 올바른지 확인
>
> 2. **엔트리 파일의 export:**
>    - `src/index-client.js` --- 클라이언트 측 export 목록
>    - `src/index-server.js` --- 서버 측 export 목록
>    - `something`이 실제로 export되고 있는지 확인
>
> 3. **타입 정의:**
>    - `src/index.d.ts` 또는 `types/index.d.ts`에서 타입 export 확인
>    - TypeScript의 타입 해결과 런타임 해결이 다를 수 있음
>
> 4. **빌드 도구 설정:**
>    - Vite/Rollup의 resolve 조건이 올바른지 확인
>    - `browser` vs `default` 조건 분기
>
> 5. **vitest.config.js의 alias:**
>    - 테스트 환경에서는 별도 alias가 적용됨
>    - `pkg.exports`에서 올바른 경로를 resolve하는지 확인
>
> 6. **해결 방법:**
>    - export가 없다면: 해당 함수를 엔트리 파일에서 export 추가
>    - 환경 분기 문제라면: browser/default 조건 확인
>    - 빌드 캐시 문제라면: `node_modules/.vite` 삭제 후 재시도

> [!summary]- 학습 포인트 요약
> - **전체 파이프라인**: .svelte --> 컴파일러(Parse/Analyze/Transform) --> JavaScript --> 번들러 --> 브라우저 --> mount/hydrate --> 인터랙티브 앱
> - **서버/클라이언트 분기**: 컴파일 시 `generate` 옵션으로 결정, 런타임에서 다른 코드 경로
> - **DEV vs Production**: `if (DEV)` 블록으로 개발 모드 전용 경고/에러, 프로덕션에서 tree-shake
> - **exports 맵**: `package.json`의 `exports` 필드가 모든 import 경로의 진입점
> - **새 기능 추가 체크리스트**: 파서 --> 분석기 --> 변환기 --> 런타임 --> 타입 --> 테스트 --> 문서 --> changeset
> - **디버깅 핵심**: 에러 메시지 텍스트로 소스 검색, 호출 스택 역추적, 테스트 샘플 참조

## Related Modules

- [[System-Architecture]] --- 전체 아키텍처 개요
- [[Compiler-Parse]] --- 파서
- [[Compiler-Analyze]] --- 분석기
- [[Compiler-Transform]] --- 변환기
- [[Reactivity-System]] --- 반응성 시스템
- [[Client-Runtime]] --- 클라이언트 런타임
- [[Server-Runtime]] --- 서버 런타임
- [[Build-System]] --- 빌드 시스템
- [[CI-CD]] --- CI/CD
- [[Compiler-Exercises]] --- 컴파일러 연습문제
- [[Runtime-Exercises]] --- 런타임 연습문제
- [[Reactivity-Exercises]] --- 반응성 연습문제
