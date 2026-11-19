---
module: exercises
path: 10-Exercises
keywords: practice, onboarding, runtime, mount, DOM
tags: [arch-runtime, practice]
---

# 런타임 --- 온보딩 연습문제

> [!info] 연습 가이드
> 아래 문제들을 통해 Svelte 5의 클라이언트/서버 런타임에 대한 이해를 점검하세요. `[trace]`는 코드 추적, `[debug]`는 디버깅, `[config]`는 설정 변경, `[extend]`는 확장 과제입니다.

## 연습 1 [trace]: mount() 호출의 내부 동작

`mount(App, { target: document.body })` 호출 시 내부에서 일어나는 일을 단계별로 추적하세요.

> [!answer]- 정답 보기
> **1. mount 함수 진입** (`src/internal/client/render.js`)
> - options에서 target, props, context 등을 추출
> - anchor 노드 결정 (target의 첫 자식 또는 지정된 anchor)
>
> **2. 컴포넌트 컨텍스트 생성**
> - `component_context` 스택에 새 컨텍스트 push
> - 부모 컨텍스트, props, lifecycle 콜백 저장소 초기화
>
> **3. 반응성 루트 생성** (`effect_root`)
> - 컴포넌트의 최상위 반응성 스코프 생성
> - 이 스코프 안에서 모든 `$state`, `$derived`, `$effect`가 관리됨
>
> **4. 컴포넌트 함수 실행**
> - 컴파일된 컴포넌트 함수 호출
> - `$state` 초기화 --> 반응성 소스 생성
> - 템플릿 함수 실행 --> DOM 노드 생성
> - 이벤트 리스너 바인딩
> - `$effect` 등록 (아직 실행하지 않음)
>
> **5. DOM 삽입**
> - 생성된 DOM 트리를 target에 삽입
> - `render_effect`들이 실행되어 동적 바인딩 활성화
>
> **6. Lifecycle 실행**
> - `onMount` 콜백들이 `user_effect`로 스케줄되어 실행
> - 마운트 완료

## 연습 2 [trace]: {#each} 블록의 DOM 업데이트

`{#each items as item (item.id)}` 블록에서 아이템이 추가/삭제될 때 DOM은 어떻게 업데이트되는지 추적하세요.

> [!answer]- 정답 보기
> **키(key) 기반 reconciliation:**
>
> 1. **배열 변경 감지:**
>    - `items`가 반응성 소스이므로, 값이 변경되면 each 블록의 `render_effect`가 트리거
>
> 2. **키 매칭:**
>    - 이전 아이템 목록과 새 아이템 목록을 `item.id` 키로 비교
>    - 세 가지 경우를 식별: 유지, 추가, 삭제
>
> 3. **아이템 추가:**
>    - 새 키에 대해 블록 인스턴스 생성
>    - 새 DOM 노드를 올바른 위치에 삽입
>    - 반응성 스코프 및 이벤트 리스너 설정
>
> 4. **아이템 삭제:**
>    - 삭제된 키의 블록 인스턴스 정리
>    - DOM 노드 제거
>    - transition이 있으면 outro 실행 후 제거
>    - 반응성 스코프 및 이벤트 리스너 해제
>
> 5. **순서 변경:**
>    - DOM 노드를 실제로 이동 (insertBefore 등)
>    - `animate:flip` 등이 있으면 FLIP 애니메이션 적용
>
> **키가 없는 경우** (`{#each items as item}`):
> - 인덱스 기반으로 기존 블록을 재사용
> - 길이가 줄면 끝에서부터 블록 제거
> - 길이가 늘면 끝에 새 블록 추가

## 연습 3 [debug]: 언마운트 후 이벤트 리스너 누수

컴포넌트가 언마운트된 후에도 이벤트 리스너가 남아있어 메모리 누수가 발생한다면, 어디를 확인해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **확인 순서:**
>
> 1. **onMount 클린업 확인:**
>    ```js
>    onMount(() => {
>      window.addEventListener('resize', handler);
>      return () => window.removeEventListener('resize', handler);
>      // ^^^^ 클린업 함수를 반환하고 있는가?
>    });
>    ```
>
> 2. **$effect 클린업 확인:**
>    ```js
>    $effect(() => {
>      el.addEventListener('scroll', handler);
>      return () => el.removeEventListener('scroll', handler);
>      // ^^^^ 반환 함수로 정리하고 있는가?
>    });
>    ```
>
> 3. **이벤트 위임 vs 직접 바인딩:**
>    - Svelte의 `on:click`은 내부적으로 관리되어 자동 정리됨
>    - `addEventListener`를 직접 호출한 경우 수동 정리 필요
>
> 4. **소스 코드 추적:**
>    - `src/internal/client/dom/elements/events.js` --- 이벤트 바인딩/해제 로직
>    - `src/internal/client/reactivity/effects.js` --- effect 소멸 시 클린업 호출
>    - `src/internal/client/render.js` --- unmount 시 전체 정리 과정
>
> 5. **디버깅 도구:**
>    - Chrome DevTools > Memory > Heap Snapshot으로 누수된 리스너 확인
>    - `getEventListeners(element)`로 요소에 바인딩된 리스너 확인

## 연습 4 [config]: SSR 렌더링 옵션 변경

서버 사이드 렌더링(SSR)의 동작을 변경하려면 어디를 수정해야 하는지 설명하세요. 예를 들어, SSR에서 특정 컴포넌트를 건너뛰고 싶다면?

> [!answer]- 정답 보기
> **관련 파일:**
>
> 1. **컴파일러 옵션** (`src/compiler/types/`)
>    - `generate: 'server'` 옵션이 SSR 코드 생성을 제어
>    - 컴파일 시 서버용 출력과 클라이언트용 출력이 다름
>
> 2. **서버 런타임** (`src/internal/server/`)
>    - `index.js` --- 서버 측 렌더링 진입점
>    - 서버에서의 컴포넌트 렌더링 로직
>    - HTML 문자열 생성 과정
>
> 3. **서버 공개 API** (`src/index-server.js`)
>    - `onMount` 등 클라이언트 전용 API의 서버 측 noop 구현
>
> 4. **서버 렌더 함수** (`svelte/server`의 `render`)
>    - `render(Component, { props })` 형태로 호출
>    - HTML 문자열과 head 내용을 반환
>
> **특정 컴포넌트 건너뛰기:**
> - 컴포넌트 내에서 `import { browser } from '$app/environment'` 등으로 환경 감지
> - `{#if browser}` 패턴으로 클라이언트 전용 렌더링
> - 또는 동적 import로 클라이언트에서만 컴포넌트 로드

## 연습 5 [extend]: 새로운 빌트인 트랜지션 추가

`svelte/transition`에 새로운 빌트인 트랜지션 (예: `rotate`)을 추가하려면 어떤 파일들을 만들고 수정해야 하는지 설명하세요.

> [!answer]- 정답 보기
> **1. 트랜지션 함수 구현** (`src/transition/index.js`)
> ```js
> export function rotate(node, { duration = 400, easing = linear, degrees = 360 }) {
>   return {
>     duration,
>     easing,
>     css: (t) => `transform: rotate(${t * degrees}deg)`
>   };
> }
> ```
>
> **2. 타입 정의 추가** (`src/transition/public.d.ts` 또는 관련 `.d.ts`)
> ```ts
> export interface RotateParams {
>   delay?: number;
>   duration?: number;
>   easing?: EasingFunction;
>   degrees?: number;
> }
> export function rotate(node: Element, params?: RotateParams): TransitionConfig;
> ```
>
> **3. 타입 재생성**
> ```bash
> cd packages/svelte && pnpm generate:types
> ```
>
> **4. 테스트 추가**
> - `tests/runtime-runes/samples/transition-rotate/` 디렉토리 생성
> - `main.svelte` --- 트랜지션을 사용하는 테스트 컴포넌트
> - `_config.js` --- 테스트 설정 (props, 기대 동작)
>
> **5. export 확인** (`package.json`)
> - `svelte/transition` export가 이미 `src/transition/index.js`를 가리키므로, 해당 파일에서 export하면 자동으로 사용 가능

## 연습 6 [trace]: 하이드레이션 과정

서버에서 렌더링된 HTML이 클라이언트에서 `hydrate()`를 통해 활성화되는 과정을 추적하세요.

> [!answer]- 정답 보기
> **1. 서버:** `render(App, { props })` 실행
> - HTML 문자열 생성 (정적 마크업 + 하이드레이션 마커)
> - 마커: 주석 노드, 데이터 속성 등으로 동적 부분 표시
>
> **2. 클라이언트:** `hydrate(App, { target })` 호출
> - mount와 유사하지만, DOM을 **새로 생성하지 않고** 기존 DOM을 **재활용**
>
> **3. DOM 워킹:**
> - 서버가 생성한 마커를 따라 기존 DOM 노드를 순회
> - 각 노드를 컴포넌트의 반응성 시스템에 연결
> - 텍스트 노드, 속성 등을 반응성 소스와 바인딩
>
> **4. 이벤트 바인딩:**
> - 서버 HTML에는 이벤트 리스너가 없으므로, 클라이언트에서 모든 이벤트 리스너 부착
>
> **5. 반응성 활성화:**
> - `$state`, `$derived` 등의 반응성 소스 초기화
> - `$effect`, `onMount` 등의 부수 효과 실행
> - 이 시점부터 완전한 인터랙티브 앱이 됨

## Related Modules

- [[Client-Runtime]] --- 클라이언트 런타임 상세
- [[Server-Runtime]] --- 서버 런타임 상세
- [[Public-API]] --- mount, hydrate, unmount API
- [[Compiler-Exercises]] --- 컴파일러 연습문제
- [[Reactivity-Exercises]] --- 반응성 연습문제
- [[Integration-Exercises]] --- 통합 연습문제
