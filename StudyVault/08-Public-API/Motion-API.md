---
module: motion
path: packages/svelte/src/motion
keywords: spring, tweened, animation, transition, easing
tags: [module-motion]
---

# Motion API

> [!info] 학습 목표
> 애니메이션, 트랜지션, 이징 기능을 제공하는 모션 모듈을 이해한다.

**난이도:** ★★

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `src/motion/spring.js` | Spring 물리 기반 애니메이션 |
| `src/motion/tweened.js` | Tweened 보간 애니메이션 |
| `src/motion/utils.js` | 공유 유틸 (is_date 등) |
| `src/motion/index.js` | 모듈 공개 엔트리포인트 |
| `src/easing/index.js` | 이징 함수 모음 |
| `src/transition/index.js` | 트랜지션 함수 (fade, fly 등) |
| `src/animate/index.js` | animate 디렉티브 구현 |

## 모듈 구조

```
svelte/motion, svelte/transition, svelte/easing, svelte/animate
+----------------------------------------------------------+
|                                                          |
|  Motion (svelte/motion)                                  |
|  +------------------------+  +------------------------+  |
|  | Spring                 |  | Tweened                |  |
|  | - 물리 시뮬레이션      |  | - 시간 기반 보간       |  |
|  | - stiffness, damping   |  | - duration, easing     |  |
|  | - tick_spring()        |  | - get_interpolator()   |  |
|  +----------+-------------+  +----------+-------------+  |
|             |                           |                |
|             +-----> writable store <----+                |
|             +-----> raf loop       <----+                |
|                                                          |
|  Transition (svelte/transition)                          |
|  +----------------------------------------------------+  |
|  | fade | fly | slide | scale | blur | draw | crossfade| |
|  +----------------------------------------------------+  |
|                                                          |
|  Easing (svelte/easing)                                  |
|  +----------------------------------------------------+  |
|  | linear | cubic | quad | quart | quint | sine | expo | |
|  | circ | elastic | back | bounce (+ In/Out/InOut)     | |
|  +----------------------------------------------------+  |
|                                                          |
|  Animate (svelte/animate)                                |
|  +----------------------------------------------------+  |
|  | flip (First-Last-Invert-Play)                       | |
|  +----------------------------------------------------+  |
+----------------------------------------------------------+
```

## Spring: 물리 기반 애니메이션

Spring은 **감쇠 스프링 물리 시뮬레이션**을 사용하여 값을 자연스럽게 애니메이션한다.

### tick_spring 알고리즘

```
acceleration = stiffness * (target - current) - damping * velocity
new_position = current + (velocity + acceleration) * dt
```

`spring.js`의 `tick_spring` 함수는 재귀적으로 동작한다:
- 숫자/Date: 직접 물리 계산
- 배열: 각 요소별 재귀
- 객체: 각 키별 재귀

> [!note] 정밀도 기반 정지
> `Math.abs(d) < precision && Math.abs(delta) < precision`이면 목표값에 도달한 것으로 판단하고 `settled = true`를 유지한다. 그렇지 않으면 `settled = false`로 설정하여 루프를 계속 돌린다.

### Spring 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `stiffness` | 0.15 | 강성 (높을수록 빠르게 수렴) |
| `damping` | 0.8 | 감쇠 (높을수록 진동 감소) |
| `precision` | 0.01 | 수렴 판정 임계값 |

### 사용 방법 (Svelte 5 Runes)

```svelte
<script>
  import { Spring } from 'svelte/motion';
  const coords = new Spring({ x: 0, y: 0 });
</script>
```

## Tweened: 시간 기반 보간 애니메이션

Tweened는 **시간과 이징 함수**를 사용하여 값을 보간한다.

### get_interpolator 로직

`tweened.js`의 `get_interpolator` 함수는 타입에 따라 보간 전략을 선택한다:

| 타입 | 보간 방식 |
|------|-----------|
| 숫자 | 선형 보간 `a + t * (b - a)` |
| Date | 타임스탬프 기준 선형 보간 |
| 배열 | 각 요소별 재귀 보간 |
| 객체 | 각 키별 재귀 보간 |
| 동일 값 | 상수 함수 반환 |

> [!warning] 타입 불일치
> `a`와 `b`의 타입이 다르거나 배열/비배열이 혼합되면 `'Cannot interpolate values of different type'` 에러가 발생한다.

### Tweened 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `delay` | 0 | 시작 전 대기 시간 (ms) |
| `duration` | 400 | 애니메이션 지속 시간 (ms) |
| `easing` | linear | 이징 함수 |
| `interpolate` | 자동 | 커스텀 보간 함수 |

## 공통 인프라

Spring과 Tweened 모두 다음 내부 모듈을 공유한다:

| 모듈 | 역할 |
|------|------|
| `internal/client/loop.js` | `loop()` --- rAF 기반 애니메이션 루프 |
| `internal/client/timing.js` | `raf` --- requestAnimationFrame 래퍼 |
| `store/shared/index.js` | `writable` --- 값 저장 및 구독 |
| `internal/client/reactivity/sources.js` | `state`, `set` --- Runes 반응성 소스 |

## Easing 함수

`svelte/easing`에서 제공하는 이징 함수들:

```
linear
  |
  +--> In / Out / InOut 변형:
       quad, cubic, quart, quint, sine,
       expo, circ, elastic, back, bounce
```

모든 이징 함수는 `(t: number) => number` 시그니처를 따른다. `t`는 0에서 1 사이의 진행률이다.

## Transitions

`svelte/transition`에서 제공하는 트랜지션 함수들:

| 트랜지션 | 설명 | 주요 옵션 |
|----------|------|-----------|
| `fade` | 투명도 변환 | duration, delay, easing |
| `fly` | 위치 + 투명도 | x, y, duration, easing |
| `slide` | 높이 슬라이드 | duration, easing, axis |
| `scale` | 크기 변환 | start, duration, easing |
| `blur` | 블러 효과 | amount, duration, easing |
| `draw` | SVG 패스 그리기 | duration, delay, easing |
| `crossfade` | 두 요소 간 교차 | duration, delay, easing, fallback |

## Animate Directive

`svelte/animate`의 `flip` (First-Last-Invert-Play) 함수는 `{#each}` 블록에서 요소 재정렬 시 부드러운 애니메이션을 제공한다.

```svelte
<script>
  import { flip } from 'svelte/animate';
</script>

{#each items as item (item.id)}
  <div animate:flip={{ duration: 300 }}>
    {item.name}
  </div>
{/each}
```

## 테스팅
- 실행: `pnpm test motion`
- 패턴: 단위 테스트 (Vitest)
- 테스트 위치: `packages/svelte/tests/motion/`

## Related Notes

- [[Public-API]] --- 전체 공개 API 개요
- [[Store-API]] --- Spring/Tweened의 기반이 되는 Store 시스템
- [[Client-Runtime]] --- rAF 루프와 반응성 효과
- [[Practice-Public-API]] --- 연습문제
