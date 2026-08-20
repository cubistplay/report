# Implement-04 — dynamic evidence action 단계 분리

## 개발 요청

`DynamicEvidenceExecutor.answer()`에 planner action 기록, memory dependency 추적,
`answer`·`ask`·잘못된 action 처리, 성공·실패 결과 생성이 한 loop에 섞여 있습니다.
action별 실행 경로와 실행 상태를 분리해 주세요.

## 완료 조건

- `answer`, `ask`, 지원하지 않는 action이 각각 독립된 실행 단계를 사용합니다.
- action·step·memory dependency 기록은 하나의 state 객체에서 관리합니다.
- 같은 memory dependency는 한 번만 기록됩니다.
- 잘못된 action도 실패 전 action log에 남습니다.
- 기존 dynamic 실행의 결과 JSON은 변경 전과 같아야 합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/decomposition.py`, `tests/test_decomposition*`에 한정합니다.
- planner protocol, output schema, memory recovery 정책은 바꾸지 않습니다.

## 작업 단위

이번 작업은 action name을 실행 Strategy로 연결하는 `DynamicActionStageRegistry`와,
실행 중 누적되는 정보를 소유하는 `DynamicEvidenceState`를 도입하는 구조 개선입니다.
