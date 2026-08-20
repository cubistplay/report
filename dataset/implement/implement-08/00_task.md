# Implement-08 — routing Policy 분해

## 개발 요청

`BrainwashRouter.route()`의 behavior, broad/domain, fact, mixed batch 분기가 한 메서드에
집중돼 있습니다. routing 우선순위와 공통 `AlgorithmPlan` 필드를 유지하면서 각 lane의 선택
정책을 분리해 주세요.

## 완료 조건

- behavior → broad/domain → fact → mixed의 기존 precedence를 유지합니다.
- 각 Policy는 자신이 담당하지 않는 batch에서 다음 Policy로 넘깁니다.
- stats, thresholds, warnings, 공통 `AlgorithmPlan` 생성은 `RoutingContext`에서 한 번만 처리합니다.
- custom Policy가 동일한 batch stats를 받아 기존 Policy 앞에 삽입될 수 있어야 합니다.
- 기존 fact·behavior route와 Update DB 회귀 test를 통과합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/router.py`, `tests/test_router.py`에 한정합니다.
- algorithm, control mode, fallback, warning, stats payload의 기존 결과 계약을 바꾸지 않습니다.

## 작업 단위

이번 작업은 batch routing의 하나의 제어 흐름을 Policy Strategy와 immutable context로 분해하는
behavior-preserving refactor입니다. 300줄을 넘지만 네 routing lane, common plan factory,
constructor injection, precedence test를 함께 유지한 단일 reviewable 단위입니다.
