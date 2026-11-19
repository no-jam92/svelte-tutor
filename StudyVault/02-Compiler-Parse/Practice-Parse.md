---
module: compiler-parse-practice
path: packages/svelte/src/compiler/phases/1-parse
keywords: parser, exercise, trace, debug, AST
tags: [module-parse, practice]
---

# Practice: Parse 단계 연습문제

> [!note] 학습 목표
> Parse 단계의 내부 동작을 이해하고, 코드를 추적하며, 디버깅 및 확장 방법을 익힙니다.

---

## Exercise 1 [trace]: .svelte 파일이 파싱될 때 코드가 거치는 단계를 추적하세요

다음 `.svelte` 파일이 파싱될 때 어떤 순서로 코드가 실행되는지 추적하세요:

```svelte
<script>
  let count = 0;
</script>

<button on:click={() => count++}>
  {count}
</button>

<style>
  button { color: red; }
</style>
```

> [!answer]- 정답 보기
> 1. **`compiler/index.js`의 `compile()` 함수** 호출
>    - `remove_bom()`으로 BOM 제거
>    - `validate_component_options()`로 옵션 검증
> 2. **`_parse(source)` 호출** -> `phases/1-parse/index.js`
>    - `Parser` 클래스 생성자 실행
>    - `regex_lang_attribute`로 `<script>` 태그의 `lang` 속성 확인 (여기서는 TS 아님)
> 3. **`state/fragment.js`** 에서 시작
>    - `<` 문자 발견 -> `state/element.js`로 전환
>    - `<script>` 태그 인식 -> `read/script.js` 호출
> 4. **`read/script.js`** -> `acorn.js`의 `parse()` 호출
>    - Acorn이 `let count = 0;`을 JavaScript AST로 파싱
>    - 결과가 `parsed.instance`에 저장됨
> 5. 다시 **`state/fragment.js`**로 돌아옴
>    - `<button>` 발견 -> `state/element.js`로 전환
>    - `on:click` 속성 파싱, `{() => count++}` 표현식은 `read/expression.js` -> Acorn으로 파싱
>    - `{count}` -> `state/tag.js`에서 ExpressionTag로 파싱
> 6. **`<style>` 블록** 발견 -> `read/style.js`
>    - CSS를 파싱하여 `parsed.css`에 저장
> 7. **`AST.Root` 객체 반환**: `{ fragment, instance, module: null, css, options: null }`

---

## Exercise 2 [trace]: `<script>` 태그의 JavaScript는 어떻게 파싱되나요?

`<script lang="ts">` 태그가 있을 때, TypeScript 코드가 최종 JavaScript AST로 변환되는 과정을 설명하세요.

> [!answer]- 정답 보기
> 1. **`Parser` 생성자**에서 `regex_lang_attribute`를 통해 `lang="ts"` 감지
>    - `this.ts = true`로 설정되고, `parsed.metadata.ts = true` 기록
> 2. **`read/script.js`** 에서 `<script>` 블록의 내용을 추출
> 3. **`acorn.js`** 에서 Acorn + **acorn-typescript 플러그인**을 사용하여 파싱
>    - TypeScript 구문 (타입 어노테이션, 인터페이스 등)을 포함한 AST 생성
> 4. 파싱 완료 후 `compile()` 함수로 돌아와서 `parsed.metadata.ts`가 `true`인지 확인
> 5. **`remove_typescript_nodes()`** 호출
>    - `parsed.fragment`, `parsed.instance`, `parsed.module` 각각에 대해 실행
>    - 타입 어노테이션, 인터페이스, 타입 별칭, `as` 표현식 등 TS 전용 노드 제거
> 6. 결과: 순수 JavaScript AST만 남아 다음 단계(analyze)로 전달

---

## Exercise 3 [config]: 파서 옵션을 변경하려면 어디를 수정해야 하나요?

`parse()` 함수에 전달할 수 있는 옵션과, 파서 동작을 커스터마이즈하는 방법을 설명하세요.

> [!answer]- 정답 보기
> **공개 API 옵션** (`compiler/index.js`의 `parse()` 함수):
>
> | 옵션 | 설명 |
> |------|------|
> | `modern` | `true`이면 Svelte 5+ 모던 AST 반환, `false`이면 레거시 AST 반환 |
> | `loose` | `true`이면 에러가 있어도 가능한 한 AST를 반환 (v5.13.0+) |
> | `filename` | (미사용, Svelte 6에서 제거 예정) |
>
> **컴파일 옵션** (`compile()` 함수):
> - `CompileOptions`에서 설정 -> `validate_component_options()`로 검증
> - `<svelte:options>` 태그의 속성도 `read/options.js`에서 파싱되어 병합됨
>
> **수정 위치:**
> - 파서 옵션 추가: `phases/1-parse/index.js`의 `Parser` 클래스 생성자
> - 컴파일 옵션 검증: `compiler/validate-options.js`
> - Acorn 파서 설정: `phases/1-parse/acorn.js`

---

## Exercise 4 [debug]: 파서 에러가 발생하면 어디를 확인해야 하나요?

사용자가 잘못된 `.svelte` 구문을 작성했을 때 에러가 어떻게 생성되고 보고되는지 추적하세요.

> [!answer]- 정답 보기
> **에러 발생 경로:**
>
> 1. **`compiler/errors.js`** - 모든 컴파일러 에러 코드가 정의됨
>    - 파서에서 `import * as e from '../../errors.js'`로 가져와 사용
>    - 예: `e.expected_token(...)`, `e.unexpected_eof(...)` 등
>
> 2. **`Parser` 클래스 내부 메서드들** (주요 에러 발생 지점):
>    - `state/element.js` - 닫히지 않은 태그, 잘못된 속성 구문
>    - `state/tag.js` - 잘못된 `{#if}`, `{#each}` 구문
>    - `read/expression.js` - 유효하지 않은 JavaScript 표현식
>    - `read/script.js` - `<script>` 블록 내 구문 오류 (Acorn이 던짐)
>
> 3. **`loose` 모드**에서는 에러 대신 가능한 한 파싱을 계속 진행
>    - `this.loose`가 `true`이면 일부 에러를 무시하고 불완전한 AST 반환
>
> **디버깅 순서:**
> 1. 에러 메시지에서 에러 코드 확인
> 2. `errors.js`에서 해당 코드 검색
> 3. 해당 에러를 호출하는 파서 코드 위치 확인
> 4. `Parser.index` 값을 로깅하여 파서가 멈춘 소스 위치 파악

---

## Exercise 5 [extend]: 새로운 템플릿 구문을 추가하려면 어떻게 해야 하나요?

가상의 `{#match expression}` 블록을 Svelte 템플릿에 추가한다고 가정할 때, 파서에서 수정해야 할 파일과 단계를 설명하세요.

> [!answer]- 정답 보기
> **수정이 필요한 파일들:**
>
> 1. **`state/tag.js`** - 새로운 블록 태그 인식
>    - `{#match`를 감지하는 분기 추가
>    - `{:case}`(가상의 하위 블록)도 여기서 처리
>    - `{/match}` 닫기 태그 처리
>
> 2. **`utils/create.js`** - `MatchBlock` AST 노드 생성 헬퍼 추가
>
> 3. **타입 정의** (`compiler/types/` 디렉토리)
>    - `AST.MatchBlock` 타입 정의 추가
>    - `AST.TemplateNode` 유니온 타입에 `MatchBlock` 포함
>
> 4. **`errors.js`** - 관련 에러 메시지 추가
>    - 예: `match_block_invalid_expression`, `match_block_unclosed` 등
>
> 5. **후속 단계도 수정 필요:**
>    - `phases/2-analyze/visitors/` - `MatchBlock.js` 분석 visitor 추가
>    - `phases/3-transform/client/visitors/` - 클라이언트 변환 visitor 추가
>    - `phases/3-transform/server/visitors/` - 서버 변환 visitor 추가
>
> **핵심 포인트:** 파서는 구문을 인식하고 AST 노드를 생성하는 역할만 합니다. 실제 동작은 analyze와 transform 단계에서 구현합니다.

---

## Exercise 6 [trace]: `loose` 모드 파싱은 일반 파싱과 어떻게 다른가요?

`parse(source, { loose: true })`를 호출했을 때 파서의 동작이 어떻게 달라지는지 설명하세요.

> [!answer]- 정답 보기
> **`loose` 모드** (v5.13.0에서 추가):
>
> - `Parser` 클래스의 `this.loose` 속성이 `true`로 설정됨
> - 일반 모드에서는 구문 오류 시 `e.*` 에러를 던져 파싱이 즉시 중단됨
> - `loose` 모드에서는 에러 대신 가능한 한 파싱을 계속 진행:
>   - 닫히지 않은 태그를 자동으로 닫기
>   - 잘못된 표현식을 건너뛰기
>   - 불완전한 블록을 부분적으로 파싱
> - **용도:** IDE/에디터의 자동완성, 구문 강조 등에서 불완전한 코드도 파싱해야 할 때 사용
> - **주의:** `loose` 모드로 생성된 AST는 `compile()`에 사용할 수 없음. `parse()` 공개 API에서만 사용 가능

---

## Related Concepts

- [[Compiler-Parse]] - Parse 단계 상세 설명
- [[Compiler-Analyze]] - AST를 분석하는 다음 단계
- [[Compiler-Transform]] - 코드 생성 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인
