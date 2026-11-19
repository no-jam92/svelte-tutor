---
module: devops-build
path: packages/svelte
keywords: rollup, build, pnpm, monorepo, types
tags: [module-devops, config-build]
---

# Build System

> [!info] 학습 목표
> Svelte 패키지의 빌드 시스템과 모노레포 구성을 이해한다.

**난이도:** ★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `packages/svelte/rollup.config.js` | 컴파일러 UMD 번들 빌드 |
| `pnpm-workspace.yaml` | pnpm 워크스페이스 정의 |
| `packages/svelte/package.json` | 패키지 exports, scripts |
| `packages/svelte/scripts/` | 빌드 관련 스크립트 |
| `.changeset/` | Changeset 기반 버저닝 설정 |

## 모노레포 구조

```
svelte-tutor/                 (루트)
|
+-- pnpm-workspace.yaml       워크스페이스 정의
+-- package.json               루트 scripts (test, lint, build)
+-- vitest.config.js           공유 테스트 설정
|
+-- packages/
|   +-- svelte/                메인 Svelte 패키지
|   |   +-- src/               소스 코드
|   |   +-- compiler/          빌드된 컴파일러 (UMD)
|   |   +-- types/             생성된 타입 정의
|   |   +-- rollup.config.js   Rollup 설정
|   |   +-- package.json       패키지 설정 + exports map
|   |
|   +-- svelte/sites/          문서 사이트 등
|
+-- .changeset/                버저닝 설정
+-- .github/workflows/         CI/CD
```

## Rollup 설정 (컴파일러 빌드)

`rollup.config.js`는 **컴파일러만** UMD 번들로 빌드한다:

```js
export default defineConfig({
  input: 'src/compiler/index.js',
  output: {
    file: 'compiler/index.js',
    format: 'umd',
    name: 'svelte'
  },
  plugins: [resolve(), commonjs(), terser()]
});
```

> [!note] 왜 컴파일러만 UMD인가?
> 컴파일러는 다양한 환경(Node.js, 브라우저 REPL, 빌드 도구)에서 사용되므로 UMD 포맷으로 번들링한다. 런타임 코드는 번들러가 tree-shaking 할 수 있도록 ES 모듈로 제공된다.

### 빌드 사이드 이펙트

```js
// 빌드 시 버전 정보를 자동 생성
import './scripts/generate-version.js';
```

Rollup 설정을 import하는 것만으로도 `generate-version.js`가 실행되어 버전 상수가 생성된다.

## pnpm 워크스페이스

pnpm 워크스페이스를 통해 모노레포 내 패키지 간 의존성을 관리한다:

- `pnpm install` --- 전체 워크스페이스 의존성 설치
- `pnpm --filter <package>` --- 특정 패키지에서만 명령 실행
- `pnpm -r` --- 모든 패키지에서 재귀 실행

## 빌드 스크립트

| 스크립트 | 설명 |
|----------|------|
| `pnpm build` | 전체 빌드 (컴파일러 번들 + 타입 생성) |
| `pnpm dev` | 개발 모드 (watch) |
| `pnpm generate:types` | dts-buddy로 타입 정의 생성 |

## 타입 생성 (dts-buddy)

Svelte는 `dts-buddy`를 사용하여 `.d.ts` 타입 파일을 생성한다:

```
src/**/*.js (JSDoc 타입)
        |
        v
   dts-buddy
        |
        v
types/*.d.ts (공개 타입 정의)
```

> [!important] 타입 변경 확인
> CI에서 빌드 후 `git status --porcelain`으로 생성된 타입이 변경되었는지 확인한다. 변경이 있으면 로컬에서 `pnpm generate:types`를 실행하고 커밋해야 한다.

## Changeset 기반 버저닝

```
1. PR에서 pnpm changeset 실행
2. .changeset/ 디렉토리에 변경 사항 기술 파일 생성
3. main 머지 후 "Version Packages" PR 자동 생성
4. 머지 시 npm publish + GitHub Release
```

| 타입 | 설명 | 예시 |
|------|------|------|
| `patch` | 버그 수정 | 0.0.x |
| `minor` | 새 기능 | 0.x.0 |
| `major` | 호환성 깨짐 | x.0.0 |

## Related Notes

- [[CI-CD]] --- CI/CD 파이프라인에서 빌드 활용
- [[Testing-Infrastructure]] --- 테스트 인프라
- [[System-Architecture]] --- 전체 프로젝트 아키텍처
