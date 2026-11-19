---
module: exercises
path: 10-Exercises
keywords: practice, onboarding, compiler
tags: [arch-compiler, practice]
---

# 컴파일러 --- 온보딩 연습문제

> [!info] 연습 가이드
> 아래 문제들을 통해 Svelte 5 컴파일러의 세 단계(Parse, Analyze, Transform)에 대한 이해를 점검하세요. `[trace]`는 코드 추적, `[config]`는 설정 변경, `[debug]`는 디버깅, `[extend]`는 확장 과제입니다.

## 연습 1 [trace]: 간단한 컴포넌트의 컴파일 과정

다음 Svelte 컴포넌트가 컴파일되는 **전체 과정**을 추적하세요:

```svelte
<script>let count = 0</script>
<button on:click={() => count++}>{count}</button>
```

각 컴파일러 단계에서 어떤 변환이 일어나는지 설명하시오.

> [!answer]- 정답 보기
> **1단계: Parse** (`src/compiler/phases/1-parse/`)
> - `<script>` 태그를 발견하고 JavaScript 부분을 acorn으로 파싱
> - `let count = 0`이 `VariableDeclaration` AST 노드가 됨
> - 템플릿 부분에서 `<button>` 엘리먼트, `on:click` 이벤트 디렉티브, `{count}` 표현식을 파싱
> - 결과: Svelte AST (Fragment > Element > Attribute, EventHandler, Expression)
>
> **2단계: Analyze** (`src/compiler/phases/2-analyze/`)
> - `count` 변수를 분석하여 반응성 상태로 표시
> - `on:click` 핸들러에서 `count++`가 상태 변경임을 감지
> - `{count}` 표현식이 반응성 의존성임을 추적
> - 스코프 분석으로 변수 바인딩 정보 수집
>
> **3단계: Transform** (`src/compiler/phases/3-transform/`)
> - `let count = 0` --> `let count = $.state(0)` (또는 `$.source(0)`)
> - `count++` --> `$.set(count, $.get(count) + 1)`
> - `{count}` --> 템플릿 내 `$.get(count)`로 반응성 읽기
> - `on:click` --> `$.event("click", button, handler)`
> - 최종 JavaScript 모듈 출력

## 연습 2 [trace]: {#if} 블록의 AST 표현

`{#if condition}...{/if}` 블록이 AST에서 어떻게 표현되고 변환되는지 추적하세요.

```svelte
{#if visible}
  <p>보입니다</p>
{:else}
  <p>숨겨져 있습니다</p>
{/if}
```

> [!answer]- 정답 보기
> **Parse 단계:**
> - `{#if`를 만나면 `IfBlock` 노드 생성 시작
> - `visible`을 condition 표현식으로 파싱
> - `<p>보입니다</p>`를 consequent (참인 경우의 자식)로 파싱
> - `{:else}`를 만나면 alternate (거짓인 경우의 자식) 파싱 시작
> - `{/if}`로 블록 종료
>
> **AST 구조:**
> ```
> IfBlock {
>   type: 'IfBlock',
>   test: Identifier { name: 'visible' },
>   consequent: Fragment { nodes: [Element <p>] },
>   alternate: Fragment { nodes: [Element <p>] }
> }
> ```
>
> **Analyze 단계:**
> - `visible`의 반응성 여부 분석
> - consequent/alternate 내부의 노드들도 재귀적으로 분석
>
> **Transform 단계:**
> - `$.if(node, () => $.get(visible), consequent_fn, alternate_fn)` 형태로 변환
> - 런타임에서 조건이 변경되면 해당 브랜치의 DOM을 생성/제거

## 연습 3 [config]: 새로운 컴파일러 경고 추가

컴파일러에 새로운 경고를 추가하려면 어떤 파일들을 수정해야 하는지 설명하세요. 예를 들어, "사용되지 않는 CSS 변수" 경고를 추가한다고 가정합니다.

> [!answer]- 정답 보기
> 수정해야 할 파일들:
>
> 1. **`src/compiler/warnings.js`** (또는 관련 에러/경고 정의 파일)
>    - 새로운 경고 코드와 메시지 템플릿 정의
>    - 예: `css_unused_variable: (name) => \`Unused CSS variable '--${name}'\``
>
> 2. **`src/compiler/phases/2-analyze/`** 내 관련 분석기
>    - CSS 관련이므로 CSS 분석 로직에서 사용되지 않는 변수를 감지하는 로직 추가
>    - `css/` 하위 파일이나 해당 visitor에서 경고 emit
>
> 3. **`src/compiler/types/`** 또는 타입 정의
>    - 새 경고 코드를 타입에 추가
>
> 4. **테스트 파일** (`tests/validator/` 또는 `tests/css/`)
>    - 경고가 올바르게 발생하는지 검증하는 테스트 케이스 추가
>    - `samples/` 디렉토리에 테스트 Svelte 파일과 기대 결과 작성
>
> 5. **문서** (선택)
>    - 경고 메시지에 대한 설명과 해결 방법 문서화

## 연습 4 [debug]: 컴파일 에러 메시지의 출처 찾기

Svelte 컴파일 중 `"Expected closing tag for <div>"` 같은 에러가 발생할 때, 이 에러 메시지의 출처를 찾으려면 어디를 확인해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **디버깅 경로:**
>
> 1. **에러 메시지 정의 검색:**
>    - `src/compiler/errors.js` 또는 유사한 에러 정의 파일에서 메시지 텍스트 검색
>    - `grep -r "Expected closing tag"` 또는 `grep -r "closing_tag"` 실행
>
> 2. **에러 발생 위치 추적:**
>    - 에러 코드/함수명을 찾은 뒤, 호출하는 곳을 역추적
>    - 이 경우 Parse 단계 (`src/compiler/phases/1-parse/`)에서 발생
>    - HTML/템플릿 파서가 여는 태그와 닫는 태그의 매칭을 검증
>
> 3. **테스트 확인:**
>    - `tests/compiler-errors/samples/`에서 해당 에러의 테스트 케이스 확인
>    - `_config.js`에서 기대되는 에러 메시지와 위치 정보 확인
>
> 4. **소스맵 활용:**
>    - 에러 객체에 `start`, `end` 위치 정보가 포함됨
>    - 원본 `.svelte` 파일의 행/열 번호로 정확한 위치 표시

## 연습 5 [extend]: 새로운 {#custom} 블록 추가

파서에 새로운 `{#custom expression}...{/custom}` 블록을 추가하는 과정을 단계별로 설명하세요.

> [!answer]- 정답 보기
> **1단계: 파서 수정** (`src/compiler/phases/1-parse/`)
> - 템플릿 파서에서 `{#custom`을 인식하는 분기 추가
> - 새로운 `CustomBlock` AST 노드 타입 정의
> - expression 파싱 및 블록 본문, `{/custom}` 종료 태그 처리
>
> **2단계: AST 타입 정의**
> - `CustomBlock` 노드의 TypeScript 인터페이스 정의
> - `type: 'CustomBlock'`, `expression`, `body` 등의 필드
>
> **3단계: Analyze 수정** (`src/compiler/phases/2-analyze/`)
> - 새 노드 타입에 대한 visitor 추가
> - expression의 반응성 분석
> - 스코프 및 바인딩 처리
>
> **4단계: Transform 수정** (`src/compiler/phases/3-transform/`)
> - `CustomBlock`을 런타임 호출로 변환하는 코드 생성기 추가
> - 클라이언트/서버 양쪽 변환 구현
>
> **5단계: 런타임 지원** (필요시)
> - `src/internal/client/`에 런타임 헬퍼 함수 추가
>
> **6단계: 테스트**
> - `tests/parser-modern/samples/`에 파서 테스트 추가
> - `tests/runtime-runes/samples/`에 런타임 테스트 추가
> - `tests/snapshot/samples/`에 스냅샷 테스트 추가

## 연습 6 [trace]: CSS 스코핑 과정

다음 코드에서 CSS가 어떻게 스코핑되는지 컴파일러 내부 과정을 추적하세요:

```svelte
<style>
  p { color: red; }
  .active { font-weight: bold; }
</style>
<p class="active">Hello</p>
```

> [!answer]- 정답 보기
> **Parse 단계:**
> - `<style>` 블록을 CSS 파서로 파싱하여 CSS AST 생성
> - 셀렉터 `p`, `.active`와 선언 블록을 구조화
>
> **Analyze 단계:**
> - 각 CSS 셀렉터가 템플릿의 어떤 요소와 매칭되는지 분석
> - `p`는 `<p>` 요소와 매칭, `.active`는 `class="active"` 속성과 매칭
> - 사용되지 않는 셀렉터가 있으면 경고 생성
>
> **Transform 단계:**
> - 고유 해시(예: `svelte-abc123`)를 생성
> - HTML 요소에 스코핑 클래스 추가: `<p class="active svelte-abc123">`
> - CSS 셀렉터에 스코핑 추가: `p.svelte-abc123 { color: red; }`
> - `.active.svelte-abc123 { font-weight: bold; }`
> - 결과: 이 컴포넌트의 스타일이 다른 컴포넌트에 영향을 주지 않음

## Related Modules

- [[Compiler-Parse]] --- 파서 상세
- [[Compiler-Analyze]] --- 분석기 상세
- [[Compiler-Transform]] --- 변환기 상세
- [[Runtime-Exercises]] --- 런타임 연습문제
- [[Reactivity-Exercises]] --- 반응성 연습문제
- [[Integration-Exercises]] --- 통합 연습문제
