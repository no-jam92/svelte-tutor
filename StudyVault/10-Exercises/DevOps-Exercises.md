---
module: exercises
path: 10-Exercises
keywords: practice, onboarding, devops, build, testing, CI
tags: [module-devops, practice]
---

# DevOps — 온보딩 연습문제

#practice #onboarding #module-devops

## 관련 모듈
- [[Build-System]]
- [[Testing-Infrastructure]]
- [[CI-CD]]

---

## 연습 1 — 빌드 프로세스 추적 [trace]
> `pnpm build`를 실행하면 내부적으로 어떤 단계들이 수행되나요? Rollup 설정부터 최종 출력물까지 추적하세요.

> [!answer]- 정답 보기
> 1. `pnpm -r --filter=./packages/* build` 실행
> 2. `packages/svelte/` 에서 `rollup -c` 실행
> 3. `scripts/generate-version.js`가 버전 파일 생성
> 4. Rollup이 `src/compiler/index.js`를 읽어 UMD 번들로 변환
> 5. `@rollup/plugin-node-resolve` → 의존성 해석
> 6. `@rollup/plugin-commonjs` → CJS 모듈 변환
> 7. `@rollup/plugin-terser` → 코드 압축
> 8. `compiler/index.js` (UMD) 출력
> 9. `pnpm generate` → 메시지 처리 + 타입 생성 (dts-buddy)

---

## 연습 2 — 테스트 스위트 탐색 [config]
> 새로운 테스트 스위트를 추가하려면 어떤 파일들을 수정해야 하나요? 예를 들어 `tests/new-feature/` 디렉토리를 추가한다면?

> [!answer]- 정답 보기
> 1. `packages/svelte/tests/new-feature/` 디렉토리 생성
> 2. `test.ts` 파일 생성 (테스트 러너 진입점)
> 3. `samples/` 디렉토리 생성 (테스트 케이스)
> 4. `vitest.config.js`에서 테스트 파일 패턴 확인 (`tests/*/test.ts` 패턴이 자동 매칭)
> 5. 필요시 `packages/svelte/tests/suite.ts`의 공통 유틸리티 활용

---

## 연습 3 — CI 파이프라인 디버깅 [debug]
> CI에서 테스트가 실패했지만 로컬에서는 통과합니다. 어떤 차이점을 확인해야 하나요?

> [!answer]- 정답 보기
> 1. `.github/workflows/ci.yml`에서 테스트 매트릭스 확인 (Node 버전, OS)
> 2. CI는 여러 Node 버전(18, 20, 22, 24)에서 테스트 — 로컬 Node 버전과 비교
> 3. CI는 Windows, macOS, Ubuntu에서 테스트 — 경로 구분자 등 OS 차이 확인
> 4. `SVELTE_NO_ASYNC=true` 환경의 TestNoAsync 잡 확인
> 5. Playwright 브라우저 버전 차이 확인 (`pnpm exec playwright install chromium`)
> 6. CI에서 `pnpm build`가 먼저 실행되는지 확인 (빌드 없이 테스트 실행 시 실패 가능)

---

## 연습 4 — 릴리스 프로세스 [trace]
> 새 버전을 배포하는 전체 과정을 추적하세요. changeset 생성부터 npm 배포까지.

> [!answer]- 정답 보기
> 1. `npx changeset` → 변경 내역 작성 (패치/마이너/메이저 선택)
> 2. `.changeset/` 에 마크다운 파일 생성
> 3. PR 머지 후 `pnpm changeset:version` 실행
> 4. changeset이 `package.json` 버전 업데이트 + CHANGELOG.md 갱신
> 5. 버전 커밋 생성 및 PR 머지
> 6. `pnpm changeset:publish` → npm 배포
> 7. `.github/workflows/release.yml`이 자동으로 이 과정을 관리

---

## 연습 5 — 타입 생성 시스템 [extend]
> Svelte 패키지에 새로운 public export를 추가할 때 타입이 올바르게 생성되도록 하려면 어떤 파일들을 수정해야 하나요?

> [!answer]- 정답 보기
> 1. 새 모듈 파일 생성 (예: `src/new-module/index.js`)
> 2. `package.json`의 `exports` 필드에 새 entry point 추가
> 3. `scripts/generate-types.js`에서 타입 생성 설정 확인
> 4. 필요시 `.d.ts` 파일 작성
> 5. `tsconfig.generated.json`에 새 경로 매핑 추가
> 6. `pnpm generate:types` 실행하여 타입 생성 확인
> 7. `scripts/check-treeshakeability.js`로 트리셰이킹 검증

---

> [!summary]- 학습 포인트 요약
> | 주제 | 핵심 사항 |
> |------|----------|
> | 빌드 시스템 | Rollup UMD 번들 + dts-buddy 타입 생성 |
> | 테스트 | Vitest 기반, `tests/*/test.ts` 패턴 자동 탐지 |
> | CI/CD | 멀티 OS/Node 매트릭스, changeset 기반 릴리스 |
> | 타입 | `generate-types.js` + `tsconfig.generated.json` |
> | 트리셰이킹 | `check-treeshakeability.js`로 번들 최적화 검증 |
