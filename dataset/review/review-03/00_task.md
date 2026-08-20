# Review-03 — benchmark provenance metadata 검토

## 검토 요청

benchmark adapter가 만든 `CorrectionRequest`에 source dataset, source case, rewrite 위치를
남기는 PR을 검토해 주세요. 초기 PR은 기존 suite를 통과하지만 provenance metadata의 ownership,
원본 row 좌표, 직렬화 계약을 직접 검증하는 test는 없습니다.

## 검토 범위

- `brainwash/benchmarks.py`의 `BenchmarkProvenance`와 `BenchmarkRequestFactory`
- adapter extra metadata와 provenance metadata의 우선순위
- MQuAKE 원본 rewrite index 및 case ID 표현
- adapter regression test의 빈 경계

## 완료 조건

- provenance가 caller extra metadata에 의해 바뀌지 않는지 확인합니다.
- skipped rewrite가 있어도 원본 rewrite index를 보존하는지 확인합니다.
- Change Request는 regression test와 별도 response commit으로 해결합니다.
- focused benchmark adapter test와 전체 suite를 통과시킵니다.

## 제한

- 변경은 `brainwash/benchmarks.py`, `tests/test_benchmark_adapters.py`에 한정합니다.
- 최초 PR은 기존 full suite를 통과하는 완결된 기능이어야 합니다.
- 리뷰 시작 뒤에는 reviewed commit을 수정하지 않고 선형 response commit만 추가합니다.
