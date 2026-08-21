# I-A1 개발 활동 보고서 — locality 평가 구조 분리

## 1. 배경

기존 `EvalHarness`는 generator 실행, 기대값·금지값 조건, locality 처리, 카운터,
실패 dict 조립을 한 메서드에서 수행했습니다. 이 구조에서는 locality를 baseline 보존
방식으로 바꾸려 할 때 평가 규칙과 결과 보고 로직을 함께 수정해야 했습니다.

이번 변경에서는 사례별 평가 규칙, evaluator 설정, 결과 집계를 각각 분리했습니다.
새 kind가 추가되어도 실행 흐름을 바꾸지 않는 구조를 목표로 했습니다.

## 2. Commit 및 PR 경계

- base: `main` / `87abaa3d5398bff678408da82e7544fe3175122b`
- Red 테스트: `418f2936453c37d066942ad97a81a7ea275bccdf`
  `test(implement-01): specify evaluator registry contracts`
- 최초 PR 및 최종 head: `ebec589e5acf7357ccbbbcc8dcd0b4f6c5b89765`
  `feat(implement-01): preserve locality with evaluation strategies`
- 최종 `main`: `ebec589e5acf7357ccbbbcc8dcd0b4f6c5b89765`

최초 head에서 다섯 가지 설계·평가 경계를 검토했습니다. 코드 결함은 발견되지 않았으므로
불필요한 Change Request나 후속 commit을 만들지 않고 최초 head를 최종 mainline으로
유지했습니다.

## 3. TDD 및 검증

Red 테스트는 아직 없는 `CaseVerdict`, `EvaluationContext`, 사용자 지정 evaluator
계약을 import하면서 실패했습니다. 새 public boundary가 아직 없음을 명확히 보여 준
실패였습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_pipeline_eval_harness -q
# Ran 6 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 74 tests — OK
```

새 테스트는 정규화한 baseline 보존, 회귀 근거, baseline이 없을 때의
`unscored`, kind별 점수, registry 사전 검증 시점, 사용자 지정 evaluator context,
behavior kind 생성을 다룹니다. 전체 테스트에서는 기존 SQLite connection 정리와
관련된 `ResourceWarning` 두 건이 표시됐지만, 모든 테스트는 통과했습니다.

## 4. 구조 개선

`CaseEvaluator` Strategy는 `EvaluationContext`를 받고 `CaseVerdict`만 반환합니다.
`ExpectedAnswerEvaluator`와 `LocalityPreservationEvaluator`가 서로 다른 규칙을
각각 담당합니다.

`EvaluatorRegistry`는 기본 evaluator 선택과 사용자 지정 registry의 coverage
검증을 담당합니다. 검증은 실행 반복문보다 먼저 수행되므로, 불완전한 설정은 generator를
호출하기 전에 실패합니다.

`EvaluationReportBuilder`는 `ScoreBreakdown`, 실패 사례, `unscored_cases`를 만듭니다.
이로써 `EvalHarness`는 검증, generator 실행, evaluator 선택에만 집중합니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 322줄 |
| 삭제 | 56줄 |
| 합계 | 378줄 |
| 파일 | 4개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/eval/harness.py`, `brainwash/eval/metrics.py`,
`tests/test_pipeline.py`, `tests/test_pipeline_eval_harness.py`입니다. 생성 파일과
formatting만 한 변경은 포함하지 않았습니다. 378줄 안에서 동작 변경, Strategy 도입,
Registry/Builder 분리, 회귀 테스트를 하나의 검토 가능한 기능으로 완결했습니다.

## 6. 리뷰 결과

리뷰에서는 `EvaluationContext`의 API 경계, registry 검증 시점,
`EvaluationReportBuilder`의 책임을 검토했습니다. 사용자 지정 evaluator 테스트,
generator 미호출 테스트, 결과 보고 테스트로 확인됐고 추가 코드 변경 없이
승인되었습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

