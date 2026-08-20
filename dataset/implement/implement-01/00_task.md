# Implement-01 — locality의 baseline 보존 검증 분리

## 개발 요청

`EvalHarness`의 locality 검증 방식을 정리해 주세요. 현재는 모델 수정 후 출력에
target text가 없는지만 확인합니다. locality는 수정 전 baseline 출력과 수정 후
출력이 같은지를 확인해야 합니다.

## 완료 조건

- locality는 정규화한 baseline 출력과 수정 후 출력을 비교합니다.
- baseline generator가 없으면 통과나 실패가 아닌 `unscored`로 남깁니다.
- reliability, generalization, `behavior`는 각 규칙과 점수를 유지합니다.
- 사용자 지정 evaluator registry에 필요한 `kind`가 없으면 generator 호출 전에 실패합니다.
- 사용자 지정 evaluator는 case, 출력, baseline 출력을 담은 `EvaluationContext`를 받습니다.
- 직접 `behavior` 요청은 `behavior` kind로 생성됩니다.

## 구현 방향

평가 규칙, evaluator 설정, 결과 집계가 `EvalHarness`에 섞이지 않게 나눕니다.
테스트를 먼저 작성하고 `test → feat` 두 commit만 남깁니다.

## 범위

`brainwash/eval/harness.py`, `brainwash/eval/metrics.py`,
`tests/test_pipeline.py`, `tests/test_pipeline_eval_harness.py`만 변경합니다.
새 model 호출, database migration, CLI option은 범위 밖입니다.
