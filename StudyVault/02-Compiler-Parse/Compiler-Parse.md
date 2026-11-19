---
module: compiler-parse
path: packages/svelte/src/compiler/phases/1-parse
keywords: parser, AST, acorn, tokenizer, template, state-machine
tags: [module-parse, arch-compiler, pattern-visitor]
---

# Compiler Parse (1-parse)

> **Purpose:** `.svelte` 파일을 AST(추상 구문 트리)로 변환하는 파서
> **Importance:** ★★★

## Overview

Parse 단계는 컴파일러 파이프라인의 첫 번째 단계입니다. 원시 `.svelte` 소스 코드 문자열을 입력받아 구조화된 `AST.Root` 객체를 출력합니다. 이 AST는 이후 [[Compiler-Analyze]] 단계와 [[Compiler-Transform]] 단계에서 사용됩니다.

## Key Files

| 파일 | 역할 |
|------|------|
| `index.js` | `Parser` 클래스 및 `parse()` 함수 내보내기 (진입점) |
| `acorn.js` | Acorn + acorn-typescript를 사용한 JavaScript/TypeScript 파싱 |
| `remove_typescript_nodes.js` | AST에서 TypeScript 전용 노드를 제거 |
| **read/** | |
| `read/script.js` | `<script>` 블록 내부의 JS/TS 코드 읽기 |
| `read/style.js` | `<style>` 블록의 CSS 파싱 |
| `read/expression.js` | `{}` 내부의 표현식 파싱 |
| `read/context.js` | 구조 분해 패턴 등 컨텍스트 읽기 |
| `read/options.js` | `<svelte:options>` 태그 파싱 |
| **state/** | |
| `state/fragment.js` | 프래그먼트(최상위 템플릿) 상태 처리 |
| `state/tag.js` | `{#if}`, `{#each}` 등 로직 태그 파싱 |
| `state/element.js` | HTML 엘리먼트 및 속성 파싱 |
| `state/text.js` | 텍스트 노드 파싱 |
| **utils/** | |
| `utils/create.js` | AST 노드 생성 헬퍼 |
| `utils/html.js` | HTML 관련 유틸리티 |
| `utils/entities.js` | HTML 엔티티 디코딩 |
| `utils/bracket.js` | 괄호 매칭 유틸리티 |
| `utils/fuzzymatch.js` | 오류 메시지용 퍼지 매칭 |

## Public Interface

컴파일러 진입점(`compiler/index.js`)에서 사용하는 주요 내보내기:

```
import { parse as _parse, Parser } from './phases/1-parse/index.js';
import { remove_typescript_nodes } from './phases/1-parse/remove_typescript_nodes.js';
import { parse_stylesheet } from './phases/1-parse/read/style.js';
```

- **`parse(source, loose?)`** - `.svelte` 소스를 파싱하여 `AST.Root` 반환
- **`Parser` 클래스** - 파서의 핵심 클래스. `Parser.forCss(source)`로 CSS 전용 파싱도 가능
- **`remove_typescript_nodes()`** - TS 노드 제거 유틸리티

## Internal Flow

```
                          .svelte 소스 코드
                                |
                                v
                    +------------------------+
                    |   Parser constructor   |
                    |  - lang 속성 감지 (ts?) |
                    |  - loose 모드 설정      |
                    +------------------------+
                                |
            +-------------------+-------------------+
            |                   |                   |
            v                   v                   v
   +-----------------+  +----------------+  +---------------+
   | <script> 파싱   |  | 템플릿 파싱     |  | <style> 파싱  |
   | (read/script)   |  | (state machine)|  | (read/style)  |
   | Acorn + TS 플러긴|  | fragment/tag/  |  | CSS 토크나이저 |
   |                 |  | element/text   |  |               |
   +-----------------+  +----------------+  +---------------+
            |                   |                   |
            v                   v                   v
    +-----------------------------------------------+
    |              AST.Root 생성                      |
    |  { fragment, instance, module, css, options }  |
    +-----------------------------------------------+
                                |
                    (if TypeScript detected)
                                |
                                v
                +-------------------------------+
                | remove_typescript_nodes()     |
                | TS 전용 구문 제거              |
                +-------------------------------+
                                |
                                v
                         AST.Root 출력
```

## Acorn을 통한 JavaScript 파싱

`<script>` 태그 내부의 JavaScript는 **Acorn** 파서를 사용하여 처리됩니다.

- `acorn.js`에서 Acorn 파서를 설정하고 **acorn-typescript** 플러그인을 통해 TypeScript 구문도 지원합니다.
- `read/script.js`가 `<script>` 블록의 시작과 끝을 찾은 후, 내부 코드를 Acorn에 전달합니다.
- `read/expression.js`는 `{expression}` 같은 템플릿 내 인라인 표현식을 Acorn으로 파싱합니다.

> [!info] Acorn 설정
> Acorn은 `ecmaVersion: 13` 이상으로 설정되며, `sourceType: 'module'`로 ES 모듈 구문을 지원합니다.

## 커스텀 템플릿 파서 (State Machine)

HTML-like 템플릿 구문은 Acorn이 아닌 **커스텀 상태 머신(state machine)** 으로 파싱합니다.

### `state/` 디렉토리 - 상태 함수들

각 파일은 파서가 특정 상태에 있을 때의 동작을 정의합니다:

| 상태 | 설명 |
|------|------|
| `fragment.js` | 최상위 진입점. 현재 문자에 따라 다른 상태로 분기 |
| `tag.js` | `{#if}`, `{#each}`, `{@html}` 등 Svelte 로직 블록 처리 |
| `element.js` | `<div>`, `<Component>` 등 HTML/컴포넌트 엘리먼트 처리 |
| `text.js` | 일반 텍스트 노드 처리 |

### `read/` 디렉토리 - 읽기 함수들

상태 함수에서 호출하여 특정 구문을 읽어옵니다:

| 함수 | 설명 |
|------|------|
| `expression.js` | `{}` 안의 JS 표현식을 Acorn으로 파싱 |
| `script.js` | `<script>` 블록 전체를 읽고 파싱 |
| `style.js` | `<style>` 블록의 CSS를 파싱 |
| `context.js` | `{#each items as item}` 등의 컨텍스트 패턴 |
| `options.js` | `<svelte:options>` 속성 파싱 |

### 동작 방식

```
fragment 상태에서 시작
  |
  +-- '<' 문자 발견 --> element 상태로 전환
  |                      +-- '<script>' --> read/script (Acorn 파싱)
  |                      +-- '<style>'  --> read/style (CSS 파싱)
  |                      +-- 일반 태그   --> 속성 파싱 후 다시 fragment
  |
  +-- '{' 문자 발견 --> tag 상태로 전환
  |                      +-- '{#if}'    --> IfBlock 노드 생성
  |                      +-- '{#each}'  --> EachBlock 노드 생성
  |                      +-- '{@html}'  --> HtmlTag 노드 생성
  |                      +-- '{expr}'   --> ExpressionTag 노드 생성
  |
  +-- 그 외 문자    --> text 상태로 전환
```

## TypeScript 제거 단계

`.svelte` 파일에 `<script lang="ts">`가 있으면 `parsed.metadata.ts`가 `true`로 설정됩니다. 컴파일러 메인 함수(`compile()`)에서 파싱 후 `remove_typescript_nodes()`를 호출하여:

- 타입 어노테이션 (`x: number`)
- 인터페이스 선언 (`interface Foo {}`)
- 타입 별칭 (`type Bar = ...`)
- 제네릭 (`<T>`)

등의 TypeScript 전용 구문을 AST에서 제거합니다. 이후 단계에서는 순수 JavaScript AST만 처리합니다.

## Dependencies

| 의존성 | 용도 |
|--------|------|
| `acorn` | JavaScript 파싱 (ECMAScript 파서) |
| `acorn-typescript` | Acorn용 TypeScript 구문 플러그인 |
| `locate-character` | 소스 위치 계산 |

## 설정

파서 동작에 영향을 주는 컴파일러 옵션:

| 옵션 | 타입 | 설명 |
|------|------|------|
| `customElement` | `boolean` | `true`일 때 커스텀 엘리먼트 모드로 파싱. `<svelte:options customElement="tag-name">` 처리 활성화 |
| `css` | `'injected' \| 'external' \| 'none'` | CSS 처리 모드. 파싱 단계에서 `<style>` 블록의 존재 여부와 처리 방식 결정 |
| `filename` | `string` | 소스 파일명. 에러 메시지의 위치 정보와 디버깅에 사용 |
| `rootDir` | `string` | 프로젝트 루트 디렉토리. 상대 경로 계산에 사용 |
| `modernAst` | `boolean` | `true`면 모던 AST 형식 출력. `false`면 레거시 AST 호환 모드 |

> [!note] `loose` 모드
> `parse(source, { loose: true })`로 호출하면 에러를 무시하고 가능한 만큼 파싱을 진행한다. IDE 자동완성 등에서 불완전한 코드를 파싱할 때 유용하다.

## Testing

```bash
# 파서 테스트 실행
pnpm test parser-modern

# 특정 테스트 파일만 실행
pnpm test parser-modern -- -t "특정 테스트 이름"
```

> [!tip] 디버깅 팁
> 파서 에러가 발생하면 `packages/svelte/src/compiler/errors.js`에서 에러 코드를 확인하세요. 대부분의 파서 에러는 `e.*` 형태로 호출됩니다.

## Related Notes

- [[Compiler-Analyze]] - 파싱된 AST를 분석하는 다음 단계
- [[Compiler-Transform]] - 최종 코드 생성 단계
- [[Compilation-Pipeline]] - 전체 컴파일 파이프라인 개요
- [[System-Architecture]] - 시스템 아키텍처 전체 구조
