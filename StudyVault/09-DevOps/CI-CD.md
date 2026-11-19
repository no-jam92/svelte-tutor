---
module: devops-ci
path: .github/workflows
keywords: CI, GitHub-Actions, release, changeset
tags: [module-devops, config-build]
---

# CI/CD

> [!info] 학습 목표
> GitHub Actions CI/CD 파이프라인과 릴리스 프로세스를 이해한다.

**난이도:** ★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `.github/workflows/ci.yml` | 메인 CI 워크플로우 |
| `.changeset/config.json` | Changeset 설정 |
| `package.json` (루트) | 스크립트 정의 |

## CI 워크플로우 구조

```
ci.yml
|
+-- trigger: push to main, pull_request
|
+-- Jobs:
    |
    +-- Tests --------> 테스트 매트릭스 (Node x OS)
    +-- TestNoAsync --> 비동기 비활성화 테스트
    +-- TSGo ---------> TypeScript Go 타입 체크
    +-- Lint ---------> 타입 체크 + 린트 + 타입 생성 검증
    +-- Benchmarks ---> 벤치마크 실행
```

## Tests Job --- 테스트 매트릭스

**전략: matrix**

| Node 버전 | OS |
|-----------|-----|
| 18 | Windows, macOS, Ubuntu |
| 20 | Ubuntu |
| 22 | Ubuntu |
| 24 | Ubuntu |

### 실행 단계

```
1. actions/checkout@v6
2. pnpm/action-setup@v4
3. actions/setup-node@v6 (+ cache: pnpm)
4. pnpm install --frozen-lockfile
5. pnpm playwright install chromium
6. pnpm test (CI=true)
```

> [!note] Playwright 설정
> `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` 환경 변수로 자동 다운로드를 비활성화하고, `pnpm playwright install chromium`으로 Chromium만 수동 설치한다.

### 타임아웃: 15분

## TestNoAsync Job

Svelte의 비동기 기능을 비활성화한 상태에서 `runtime-runes` 테스트를 실행한다.

```
SVELTE_NO_ASYNC=true pnpm test runtime-runes
```

- Node 22, Ubuntu
- 타임아웃: 10분

## TSGo Job

TypeScript Go (`@typescript/native-preview`)를 사용한 타입 체크:

```
cd packages/svelte && pnpm i -D @typescript/native-preview
cd packages/svelte && pnpm check:tsgo
```

- Node 24, Ubuntu
- 타임아웃: 5분

## Lint Job

3가지 검증을 순차적으로 수행한다 (하나가 실패해도 나머지 실행):

```
1. pnpm check         --> TypeScript 타입 체크
2. pnpm lint          --> ESLint 린트
3. pnpm build && git status --porcelain
   --> 빌드 후 생성된 타입이 커밋된 것과 일치하는지 확인
```

> [!warning] 타입 불일치 시
> `"Generated types have changed"` 에러가 발생하면, 로컬에서 `cd packages/svelte && pnpm generate:types`를 실행하고 결과를 커밋해야 한다.

## Benchmarks Job

```
pnpm bench (CI=true)
```

- Node 18, Ubuntu
- 타임아웃: 15분

## Release 프로세스 (Changeset 기반)

```
개발자                    CI/Bot                    npm
  |                         |                        |
  |  1. pnpm changeset      |                        |
  |  --> .changeset/xxx.md  |                        |
  |                         |                        |
  |  2. PR --> main 머지    |                        |
  |  ---------------------->|                        |
  |                         |  3. "Version Packages" |
  |                         |     PR 자동 생성        |
  |                         |                        |
  |  4. Version PR 머지     |                        |
  |  ---------------------->|                        |
  |                         |  5. npm publish         |
  |                         |  ---------------------->|
  |                         |  6. GitHub Release      |
  |                         |     생성                |
```

### Changeset 워크플로우

1. **변경 기술:** `pnpm changeset`으로 변경 내용과 semver 타입 (patch/minor/major) 선택
2. **파일 생성:** `.changeset/` 디렉토리에 마크다운 파일 생성
3. **자동 PR:** main 머지 후 changeset bot이 "Version Packages" PR 생성
4. **버전 업데이트:** PR 머지 시 `package.json` 버전 업데이트 + CHANGELOG 생성
5. **퍼블리시:** 자동 npm publish + GitHub Release

## PR Preview Builds

PR에서 테스트 매트릭스가 실행되어, 코드 변경이 다양한 환경에서 동작하는지 확인한다. 모든 job이 성공해야 머지가 가능하다.

## Ecosystem CI Testing

외부 프로젝트(SvelteKit 등)에서 Svelte 변경사항이 호환되는지 별도 CI로 검증한다.

## Related Notes

- [[Build-System]] --- 빌드 시스템 상세
- [[Testing-Infrastructure]] --- 테스트 인프라 상세
- [[System-Architecture]] --- 전체 프로젝트 아키텍처
