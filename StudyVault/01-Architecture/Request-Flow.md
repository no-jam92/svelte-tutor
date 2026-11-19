---
module: architecture
path: 01-Architecture
keywords: request-flow, compilation, rendering, lifecycle
tags:
  - "#arch-compiler"
  - "#arch-runtime"
  - "#pattern-dual-runtime"
  - "#api-runtime"
  - "#module-client"
  - "#module-server"
---

# Request Flow

> [!abstract] 목적
> `.svelte` 파일이 작성되어 사용자 화면에 렌더링되기까지의 전체 흐름을 추적합니다.
> CSR(Client-Side Rendering)과 SSR(Server-Side Rendering) 두 가지 경로를 모두 다룹니다.

---

## Client-Side Rendering Flow (CSR)

### End-to-End 흐름

```
  [1] 개발자가 .svelte 파일 작성
       |
       v
  [2] Vite 플러그인이 compile() 호출
       |
       v
  [3] ============= COMPILE TIME (Build) =============
       |
       +---> Phase 1: Parse
       |     소스 → AST (acorn + template tokenizer)
       |
       +---> Phase 2: Analyze
       |     AST 순회, 스코프/바인딩/의존성 분석
       |
       +---> Phase 3: Transform (client mode)
       |     AST → JavaScript 모듈 (DOM 조작 코드)
       |
       v
  [4] JavaScript 모듈 출력 (.js)
       |
       v
  [5] ============= RUN TIME (Browser) ===============
       |
       v
  [6] 브라우저가 JS 모듈 로드
       |
       v
  [7] mount(Component, { target }) 호출
       |
       v
  [8] Runtime이 컴포넌트 인스턴스 생성
       |
       +---> $state() → Source signal 생성 + Proxy 래핑
       +---> $derived() → Derived signal 생성
       +---> $effect() → Effect 등록
       |
       v
  [9] DOM 초기 렌더링 (fine-grained DOM 조작)
       |
       v
  [10] ============= INTERACTIVE (User) ==============
       |
       v
  [11] 사용자 상호작용 (클릭, 입력 등)
       |
       v
  [12] 이벤트 핸들러 실행
       |
       v
  [13] $state 값 변경 (Proxy가 변경 감지)
       |
       v
  [14] 의존하는 $derived 재계산 (lazy)
       |
       v
  [15] 관련 $effect 재실행
       |
       v
  [16] DOM 업데이트 (변경된 부분만 fine-grained update)
       |
       v
  [11] ... (반복)
```

### 핵심 파일 매핑

| 단계 | 관련 파일 | 설명 |
|------|-----------|------|
| [2] Compile 호출 | `src/compiler/index.js` | `compile()` 진입점 |
| [3] Parse | `src/compiler/phases/1-parse/index.js` | 소스 파싱 |
| [3] Analyze | `src/compiler/phases/2-analyze/index.js` | AST 분석 |
| [3] Transform | `src/compiler/phases/3-transform/index.js` | 코드 생성 |
| [7] mount | `src/index-client.js` → `src/internal/client/render.js` | 마운트 진입 |
| [8] Reactivity | `src/internal/client/reactivity/sources.js` | Signal 생성 |
| [8] Proxy | `src/internal/client/proxy.js` | 상태 Proxy 래핑 |
| [9] DOM | `src/internal/client/dom/` | DOM 조작 |
| [15] Effects | `src/internal/client/reactivity/effects.js` | Effect 재실행 |

---

## Server-Side Rendering Flow (SSR)

### End-to-End 흐름

```
  [1] 서버가 HTTP 요청 수신
       |
       v
  [2] ============= COMPILE TIME (Build or On-demand) ==========
       |
       v
  [3] compile(source, { generate: 'server' })
       |
       +---> Phase 1: Parse    (CSR과 동일)
       +---> Phase 2: Analyze  (CSR과 동일)
       +---> Phase 3: Transform (server mode)
       |     - 반응성 코드 없음
       |     - HTML 문자열 연결 코드 생성
       |
       v
  [4] Server JS 모듈 출력
       |
       v
  [5] ============= SERVER RUNTIME ==============================
       |
       v
  [6] Server Runtime이 컴포넌트 실행
       |
       +---> 상태를 정적 값으로 평가
       +---> 템플릿을 HTML 문자열로 렌더링
       +---> CSS를 인라인으로 삽입
       |
       v
  [7] 완성된 HTML 문자열 반환
       |
       v
  [8] HTTP 응답으로 HTML 전송
       |
       v
  [9] ============= CLIENT HYDRATION ============================
       |
       v
  [10] 브라우저가 HTML 수신 (즉시 렌더링 -- 빠른 FCP)
       |
       v
  [11] Client JS 모듈 로드
       |
       v
  [12] hydrate(Component, { target }) 호출
       |
       +---> 기존 DOM 노드 재사용 (새로 생성하지 않음)
       +---> 이벤트 리스너 연결
       +---> 반응성 시스템 활성화 ($state, $derived, $effect)
       |
       v
  [13] 인터랙티브 앱 완성
       |
       v
  [14] ... (이후 CSR 흐름과 동일)
```

### SSR vs CSR 비교

| 구분 | CSR | SSR + Hydration |
|------|-----|-----------------|
| 초기 화면 | JS 로드 후 렌더링 (느림) | HTML 즉시 표시 (빠름) |
| 컴파일 대상 | `generate: 'client'` | `generate: 'server'` + `generate: 'client'` |
| 서버 런타임 | 사용하지 않음 | `internal/server/renderer.js` |
| 클라이언트 진입 | `mount()` | `hydrate()` |
| SEO | JS 실행 필요 | HTML 즉시 인덱싱 가능 |

---

## ASCII Diagram: Dual Runtime Flow

```
                        .svelte source
                             |
                     +-------+-------+
                     |               |
                     v               v
              compile()         compile()
           generate:'client'  generate:'server'
                     |               |
                     v               v
              Client JS        Server JS
              Module           Module
                     |               |
                     |               v
                     |        Server Runtime
                     |        (renderer.js)
                     |               |
                     |               v
                     |          HTML string
                     |               |
                     |     +---------+---------+
                     |     |                   |
                     |     v                   v
                     |  <html>...</html>    HTTP Response
                     |     |                   |
                     |     v                   |
                     |  Browser renders        |
                     |  static HTML            |
                     |     |                   |
                     v     v                   |
                  hydrate() or mount()         |
                     |                         |
                     v                         |
              Client Runtime                   |
              +-- reactivity (signals) --------+
              +-- DOM (fine-grained ops)
              +-- events (listeners)
                     |
                     v
              Interactive App
```

---

## Reactivity Update Cycle (상세)

> [!note] mount/hydrate 이후 반응성 업데이트 사이클

```
  User Interaction
       |
       v
  Event Handler
       |
       v
  $state mutation  ──→  Proxy trap 감지
       |
       v
  Source signal 업데이트
       |
       v
  Dirty flag 전파 (의존 derived/effect에 표시)
       |
       v
  Scheduler가 batch 수집
       |
       v
  Microtask에서 flush
       |
       +──→ Derived: 재계산 (lazy, 필요 시만)
       |
       +──→ Effect: 재실행
       |         |
       |         v
       |    DOM 업데이트 (변경된 노드만)
       |
       v
  화면 갱신 완료
```

### 관련 파일

| 파일 | 역할 |
|------|------|
| `reactivity/sources.js` | Source signal, dirty 전파 |
| `reactivity/deriveds.js` | Derived 재계산, lazy evaluation |
| `reactivity/effects.js` | Effect 스케줄링 및 재실행 |
| `reactivity/batch.js` | 업데이트 배칭 (한 번에 모아서 처리) |
| `proxy.js` | Proxy trap을 통한 변경 감지 |
| `runtime.js` | 스케줄러, 마이크로태스크 flush |

---

## Related Notes

- [[System-Architecture]] -- 전체 아키텍처 개요
- [[Compilation-Pipeline]] -- 컴파일러 3단계 상세
- [[MOC]] -- 학습 경로 및 모듈 맵
- [[Quick-Reference]] -- 디버깅 팁 및 파일 위치
