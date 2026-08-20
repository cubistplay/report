# Review-05 — support retrieval trace 검토

## 검토 요청

support knowledge 조회가 답변만 반환하던 흐름에 query, source, score, reason을 남기는 trace를
추가한 PR을 검토합니다. 최초 PR은 기존 전체 test를 통과하지만 semantic 경로의 score 의미와
batch trace 계약을 직접 검증하지 않습니다.

## 검토 범위

- `brainwash/decomposition.py`의 `SupportRetrieval` Value Object
- exact·semantic·lexical 조회의 trace source와 score
- 기존 `retrieve()`의 반환 계약 보존
- batch trace와 decomposition regression test

## 완료 조건

- trace의 score가 threshold가 아니라 실제 matcher similarity인지 확인합니다.
- 기존 답변 조회 API가 trace 도입 뒤에도 같은 답을 반환하는지 확인합니다.
- Change Request는 regression test와 후속 code commit으로 해결합니다.
- focused decomposition test와 전체 suite를 통과시킵니다.

## 제한

- 변경은 `brainwash/decomposition.py`, `tests/test_decomposition.py`에 한정합니다.
- 최초 PR은 기존 full suite를 통과하는 완결된 기능이어야 합니다.
- 리뷰 시작 뒤에는 reviewed commit을 수정하지 않고 선형 response commit만 추가합니다.
