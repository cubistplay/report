# R-A5 최종 리뷰 보고서 — support retrieval trace

## 1. 검토 배경

대상 PR은 support knowledge 조회 결과를 `SupportRetrieval` Value Object로 남기고, 기존 `retrieve()`는
answer만 계속 반환하도록 확장했습니다. 초기 head는 전체 test 99건을 통과했지만 semantic trace의 score가
실제 matcher 결과인지와 batch trace의 결과 계약은 검증하지 않았습니다.

리뷰에서는 trace가 단순 log가 아니라 evaluation과 운영 분석이 신뢰할 수 있는 관측값인지, 새 API가 기존
answer 조회를 바꾸지 않는지 검토했습니다.

## 2. Commit 및 PR 경계

- base: `1b5bba293d92d665d6bfdd750e57b0b85a807ee6`
- 최초 PR branch/head: `review/review-05-support-retrieval-trace` /
  `581d54ecea1bcb4ba84dd051c75412faef75d5c4`
- 리뷰 반영 테스트: `266a5bf5496a0e73ff6ac491c2810c512efe81ae`
  `test(review-05): specify support retrieval traces`
- 리뷰 반영 수정 및 최종 head: `10a7f021c24f860fe599ed9d2040965aaf64bd02`
  `fix(review-05): preserve support retrieval confidence`

리뷰 뒤 최초 PR commit은 변경하지 않았습니다. regression test와 response code를 새 commit으로 누적해
`main`의 선형 이력을 유지했습니다.

## 3. 발견 사항과 반영 결과

| 심각도 | 발견 사항 | 영향 | 반영 |
| --- | --- | --- | --- |
| blocking | semantic trace에 threshold `0.6` 기록 | 실제 similarity를 잃어 evaluation·운영 분석이 왜곡됨 | helper가 answer와 best score 반환 |
| important | trace 및 batch result 계약 test 부재 | source·score·기존 API 호환 회귀 탐지 불가 | fixed-score·batch regression test 추가 |
| question | no-match의 score 의미 | score를 confidence로 오해할 가능성 | source와 reason으로 비교 여부를 구분 |
| question | 기존 API 위임 | answer selection 순서 변경 위험 | trace answer를 단일 출처로 사용 |
| question | batch 입력 정렬 | evaluation input/result mapping 위험 | no-match 포함 1:1 순서 보존 확인 |

## 4. 리뷰 품질과 협업

첫 Change Request는 threshold와 observed score의 역할을 코드로 구분해 설명했습니다.

```python
# threshold는 선택 기준입니다.
if similarity >= 0.6:
    answer = candidate_answer

# trace에는 선택에 사용된 실제 관측값을 남깁니다.
SupportRetrieval(..., score=similarity)
```

따라서 `0.93`으로 선택된 결과를 `0.6`으로 기록하면 단순 표현 차이가 아니라, 운영자가 결과 강도를
잘못 비교하게 됩니다. 수정 방향으로 helper의 tuple 반환과 fixed-score matcher test를 함께 제시했습니다.

[scikit-learn의 decision threshold 문서](https://scikit-learn.org/stable/modules/classification_threshold.html)도
모델 score와 threshold를 적용한 decision을 분리해 설명합니다. 이 PR의 similarity는 class probability가
아니지만, “관측 score를 보존하고 threshold는 선택 rule로만 쓴다”는 구분은 동일하므로 해당 자료와 코드
예시를 함께 제공했습니다.

두 번째 Change Request는 trace Value Object의 public contract를 test로 보호하도록 안내했습니다. semantic
trace의 source·score와 기존 `retrieve()` answer를 한 test에서, exact/no-match batch의 순서·정규화·reason을
다른 test에서 확인해 caller 관점의 regression을 남겼습니다.

no-match score와 batch 정렬은 결함으로 단정하지 않고, caller가 실제로 어떻게 해석하는지 확인하는 질문으로
다뤘습니다. 작업자는 수정 이유와 test 범위를 구체적으로 답했고, 리뷰어는 반영 commit과 검증 결과를
확인한 뒤 승인했습니다.

## 5. 검증

```bash
python3 -m unittest discover -s tests -q
# Ran 99 tests — OK  (initial PR)

python3 -m unittest tests.test_decomposition -q
# Ran 24 tests — OK

python3 -m unittest discover -s tests -q
# Ran 101 tests — OK  (final)
```

## 6. 변경 범위

| 구간 | 파일 | 추가 | 삭제 | 합계 |
| --- | --- | ---: | ---: | ---: |
| 최초 PR | `brainwash/decomposition.py` | 54 | 4 | 58 |
| 리뷰 반영 | `brainwash/decomposition.py`, `tests/test_decomposition.py` | 50 | 6 | 56 |

초기 변경량 58줄은 Review PR의 일반적인 50~200줄 범위 안입니다. support retrieval trace라는 하나의
기능을 검토했고, 반영 범위도 confidence·API compatibility·batch contract에 한정했습니다.
