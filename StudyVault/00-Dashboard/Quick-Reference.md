---
module: dashboard
path: 00-Dashboard
keywords: quick-reference, commands, debugging, setup
tags: [dashboard, quick-reference]
---

# Quick Reference

> [!info] 빠른 참조
> 자주 사용하는 명령어, 파일 위치, 디버깅 팁을 한눈에 정리한 문서입니다.
> 전체 아키텍처는 [[MOC]]를 참고하세요.

---

## Key Commands

| 작업 | 명령어 | 설명 |
|------|--------|------|
| Install | `pnpm install` | 모든 의존성 설치 |
| Build | `pnpm build` | 전체 패키지 빌드 |
| Dev | `cd packages/svelte && pnpm dev` | 개발 모드 (watch + rebuild) |
| Test | `pnpm test` | 전체 테스트 스위트 실행 |
| Lint | `pnpm lint` | ESLint + 코드 스타일 검사 |
| Format | `pnpm format` | Prettier 코드 포맷팅 |
| Bench | `pnpm bench` | 벤치마크 실행 |
| Changeset | `pnpm changeset` | 변경 로그 항목 생성 |

---

## Environment Setup

> [!warning] 사전 준비
> Node.js >= 18, pnpm >= 9가 필요합니다.

```bash
# 1. 저장소 클론
git clone <repo-url>
cd svelte-tutor

# 2. pnpm 설치 (없는 경우)
corepack enable
corepack prepare pnpm@latest --activate

# 3. 의존성 설치
pnpm install

# 4. 빌드 확인
pnpm build

# 5. 테스트 확인
pnpm test

# 6. 개발 시작
cd packages/svelte && pnpm dev
```

> [!tip] 에디터 설정
> VS Code 사용 시 ESLint, Prettier, Svelte for VS Code 확장을 설치하세요.

---

## Important File Locations

> [!note] 경로 기준
> 모든 경로는 프로젝트 루트(`svelte-tutor/`) 기준입니다.

| 구분 | 파일/디렉토리 | 설명 |
|------|---------------|------|
| **Root Config** | `package.json` | 모노레포 루트 설정 |
| **Compiler Entry** | `packages/svelte/src/compiler/index.js` | 컴파일러 공개 API 진입점 |
| **Runtime Entry (Client)** | `packages/svelte/src/index-client.js` | 클라이언트 런타임 진입점 |
| **Runtime Entry (Server)** | `packages/svelte/src/index-server.js` | 서버 런타임 진입점 |
| **Internal Client** | `packages/svelte/src/internal/client/` | 클라이언트 내부 런타임 |
| **Internal Server** | `packages/svelte/src/internal/server/` | 서버 내부 런타임 |
| **Reactivity** | `packages/svelte/src/internal/client/reactivity/` | 시그널 기반 반응성 코어 |
| **Store** | `packages/svelte/src/store/` | 스토어 API |
| **Tests** | `packages/svelte/tests/` | 테스트 스위트 |
| **Playground** | `playgrounds/` | 개발용 플레이그라운드 |
| **CI Workflows** | `.github/workflows/ci.yml` | GitHub Actions CI 설정 |
| **Release** | `.github/workflows/release.yml` | 릴리스 자동화 워크플로우 |

---

## Common Debugging Tips

| 증상 | 진입점 파일 | 설명 |
|------|-------------|------|
| **컴파일러 에러 메시지** | `packages/svelte/src/compiler/errors.js` | 모든 컴파일 에러 정의. 에러 코드로 검색 |
| **컴파일러 경고 메시지** | `packages/svelte/src/compiler/warnings.js` | 모든 컴파일 경고 정의 |
| **런타임 에러** | `packages/svelte/src/internal/client/errors.js` | 클라이언트 런타임 에러 정의 |
| **런타임 경고** | `packages/svelte/src/internal/client/warnings.js` | 클라이언트 런타임 경고 정의 |
| **개발 모드 전용 코드** | `packages/svelte/src/internal/client/dev/` | DEV 빌드에서만 실행되는 검증 로직 |
| **반응성 문제** | `packages/svelte/src/internal/client/reactivity/sources.js` | `$state` 시그널 소스 코드 |
| **이펙트 문제** | `packages/svelte/src/internal/client/reactivity/effects.js` | `$effect` 실행 로직 |
| **파생값 문제** | `packages/svelte/src/internal/client/reactivity/deriveds.js` | `$derived` 계산 로직 |
| **프록시 추적** | `packages/svelte/src/internal/client/proxy.js` | 객체/배열 상태 추적의 Proxy 구현 |
| **SSR 렌더링** | `packages/svelte/src/internal/server/renderer.js` | 서버 사이드 렌더링 코어 |
| **하이드레이션** | `packages/svelte/src/internal/client/hydratable.js` | SSR → 클라이언트 하이드레이션 로직 |
| **컴파일러 옵션** | `packages/svelte/src/compiler/validate-options.js` | 컴파일러 옵션 검증/기본값 |

> [!tip] 디버깅 전략
> 1. 에러 메시지의 코드를 `errors.js` 또는 `warnings.js`에서 검색
> 2. 해당 에러를 throw하는 위치를 역추적
> 3. 컴파일러 문제라면 Phase 1 → 2 → 3 순서로 확인
> 4. 런타임 문제라면 `reactivity/` 디렉토리부터 확인

---

## Related Notes

- [[MOC]] -- 전체 아키텍처 및 학습 경로
- [[System-Architecture]] -- 시스템 아키텍처 상세
- [[Compilation-Pipeline]] -- 컴파일러 파이프라인 상세
- [[Request-Flow]] -- 요청 흐름 추적
