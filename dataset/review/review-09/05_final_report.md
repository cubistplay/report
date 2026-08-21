# R-A9 memory trigger batch 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 검토 대상

feat(review-09): batch memory trigger decisions는 여러 prompt의 MemoryTrigger와 batch summary를 반환하는 API입니다. 최초 PR 9a9d26d은 brainwash/algorithms/memory_edit.py에 50줄을 추가했고, 기존 전체 test 126건을 통과했습니다.

## 2. 주요 검토 및 반영

### 2.1 positional batch result 보존

초기 구현은 prompt를 dict key로 사용했습니다. [Python mapping types 공식 문서](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict)의 dict는 key-value mapping이므로 같은 prompt value 두 개를 개별 결과로 보존할 수 없습니다.

    {prompt: trigger for prompt in [question, question]}
    # question key 하나만 남음

반영 후 MemoryTriggerBatch는 trigger tuple을 가지며 evaluate_many와 retrieve_many 모두 입력 순서·중복·cardinality를 보존합니다.

### 2.2 triggered count 의미 수정

초기 triggered_count는 batch result 개수였습니다. no-match도 result에는 포함되므로 memory_triggered=False인 input이 activation으로 집계되는 문제가 있었습니다. 반영 후에는 실제 memory_triggered bool을 합산하며, request_count와 blocked_count는 별도 의미를 유지합니다.

## 3. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 9a9d26d | batch trigger Value Object와 API | 전체 126건 통과 |
| 리뷰 명세 | 0258b13 | duplicate order·triggered count test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | 85b6b66 | positional result와 count semantics | runtime 15건, 전체 128건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## 4. 결론

Batch Result Value Object 구조를 유지하면서 input/result alignment와 monitoring metric의 의미를 보완했습니다. duplicate query cache는 public result contract와 분리해야 하므로 이번 변경 범위에는 포함하지 않았습니다.
