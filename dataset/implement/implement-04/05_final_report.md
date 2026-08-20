# I-A4 개발 활동 보고서 — dynamic evidence action 단계 분리

## 1. 배경

`DynamicEvidenceExecutor.answer()`는 planner action 선택 뒤 action 기록, memory
dependency 추적, step 실행, recovery, 성공·실패 결과 생성을 모두 처리하고 있었습니다.
같은 실행에서 함께 변하는 값이 여러 분기에 흩어져 있어 action을 추가하거나 기록 정책을
바꿀 때 loop 전체를 다시 확인해야 했습니다.

이번 변경은 action별 실행 경로를 Strategy로 분리하고, 실행 중 누적되는 값을
`DynamicEvidenceState`로 모았습니다. planner protocol과 결과 schema, memory recovery
정책은 유지했습니다.

## 2. Commit 및 PR 경계

- base: `main` / `3c11c0234491b57001cd620ba284071c4ad4957b`
- Red 테스트: `5e08fba28c3a741e1901c3c9c5b2b7945f8367cf`
  `test(implement-04): specify dynamic evidence stages`
- 최초 PR 및 최종 head: `f3795e270ace457f28a5a04caf9a1d47a4e989b7`
  `refactor(implement-04): decompose dynamic evidence actions`
- 최종 `main`: `f3795e270ace457f28a5a04caf9a1d47a4e989b7`

최초 head에서 action 전환, state 일관성, 결과 보존의 세 가지 경계를 검토했습니다.
코드 결함은 발견되지 않아 Change Request나 후속 commit은 만들지 않았습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 아직 없는 `DynamicActionStageRegistry`, action stage, state object import에서
실패했습니다. stage 선택, dependency 중복 제거, invalid action 기록 후 실패를 먼저
명세로 고정한 것입니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_decomposition_stages -q
# Ran 3 tests — OK

python3 -m unittest tests.test_decomposition -q
# Ran 22 tests — OK

python3 -m unittest discover -s tests -q
# Ran 86 tests — OK
```

또한 동일한 memory와 scripted planner를 변경 전 head와 현재 head에서 실행한 뒤
`DecompositionAnswer.to_dict()` 전체 JSON을 비교했습니다. `diff` 결과가 없어 action log,
plan, step 결과를 포함한 대표 dynamic 실행 결과가 보존됐음을 확인했습니다.

## 4. 구조 개선

`DynamicActionStageRegistry`는 action name을 실행 Strategy로 연결합니다.
`AnswerActionStage`와 `AskActionStage`는 각각 답변 확정과 evidence step 실행을 담당하며,
`InvalidActionStage`는 기록된 잘못된 action을 예측 없이 실패로 끝냅니다.

`DynamicEvidenceState`는 step 결과와 placeholder rendering용 answer lookup, action log,
memory dependency, memory 사용 상태를 한 실행 단위로 관리합니다. dependency 기록은 id로
중복을 제거합니다. executor는 context를 만들고 stage 실행 순서와 종료 조건만 조정합니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 329줄 |
| 삭제 | 162줄 |
| 합계 | 491줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/decomposition.py`와 `tests/test_decomposition_stages.py`입니다.
491줄은 action Strategy 도입, state object 분리, 기존 loop 제거, 회귀 테스트, 결과
동등성 검증을 한 단위로 완결하는 데 필요한 범위입니다.

## 6. 리뷰 결과

리뷰에서는 action 전환 책임, state가 소유하는 함께 변하는 기록, memory recovery와
조기 종료를 포함한 결과 보존 범위를 검토했습니다. stage/state 테스트, 기존 decomposition
테스트, 변경 전후 JSON 비교로 확인했고 추가 코드 변경 없이 승인되었습니다.
