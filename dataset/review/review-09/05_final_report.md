# R-A9 리뷰를 통한 batch result 정합성 손실 위험 제거

> Synthetic GitHub artifact: true

## 활동 요약

batch memory trigger API의 최초 구현을 검토했습니다. prompt-keyed dict가 duplicate input을 잃는 문제와
result 수를 activation 수로 잘못 집계하는 문제를 Python mapping 규칙과 no-match 사례로 재현했습니다.
positional result와 signal 기반 count를 test→fix 순서로 반영해, evaluation consumer의 input/result
alignment와 monitoring metric의 의미를 함께 바로잡았습니다.

## 1. 현황 및 이슈

`feat(review-09): batch memory trigger decisions`는 여러 prompt의 `MemoryTrigger`와 batch summary를
반환하는 API입니다. 최초 PR `5ae605a`은 `brainwash/algorithms/memory_edit.py`에 49줄을 추가했고,
기존 전체 test 126건을 통과했습니다.

batch evaluation API는 input과 output을 위치별로 연결할 수 있어야 하지만, 최초 구현은 prompt를 dict
key로 사용해 duplicate prompt를 잃었습니다. 또한 `triggered_count`가 실제 activation 수가 아닌 result
개수를 반환했습니다. 이 문제는 단위 로직보다 public result contract와 monitoring metric 의미에 영향을
주므로, cardinality·순서·지표 정의를 코드리뷰 관점에서 검토하기 위해 이 활동을 선정했습니다.

## 2. 주요 검토 및 반영

### 2.1 positional batch result 보존

초기 구현은 prompt를 dict key로 사용했습니다. [Python mapping types 공식 문서](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict)의 dict는 key-value mapping이므로 같은 prompt value 두 개를 개별 결과로 보존할 수 없습니다.

    {prompt: trigger for prompt in [question, question]}
    # question key 하나만 남음

반영 후 MemoryTriggerBatch는 trigger tuple을 가지며 evaluate_many와 retrieve_many 모두 입력 순서·중복·cardinality를 보존합니다.

### 2.2 triggered count 의미 수정

초기 triggered_count는 batch result 개수였습니다. no-match도 result에는 포함되므로 memory_triggered=False인 input이 activation으로 집계되는 문제가 있었습니다. 반영 후에는 실제 memory_triggered bool을 합산하며, request_count와 blocked_count는 별도 의미를 유지합니다.

## 3. 활동 내용

리뷰 의도는 batch Result Value Object의 형태를 바꾸는 것보다 caller가 의존할 수 있는 계약을 먼저
명확히 하는 것이었습니다. 리뷰어는 Python dict의 key uniqueness를 공식 문서와 duplicate prompt 예시로
설명하고, input order와 1:1 cardinality를 보존하는 tuple/list result를 제안했습니다. 이어
`triggered_count`가 request count와 다른 operational question에 답해야 함을 no-match 사례로 구체화했습니다.

작업자는 duplicate input의 순서 보존과 no-match batch의 count를 먼저 실패시키는 test commit을 추가했습니다.
fix commit에서는 trigger를 positional tuple로 저장하고 `evaluate_many()`·`retrieve_many()`가 같은 길이와
순서를 보존하도록 변경했습니다. `triggered_count`는 `memory_triggered` signal만 합산하고,
`request_count`·`blocked_count`는 분리해 summary field의 의미를 명확히 했습니다.

이 리뷰는 “테스트를 추가해 주세요”라는 요청을 넘어, 어떤 입력이 사라지는지와 어떤 metric이 잘못
읽히는지를 예시·문서·회귀 test로 연결했습니다. 리뷰어와 작업자가 public API의 correctness와 내부 cache
최적화를 분리해 논의한 기록도 남겼습니다.

## 4. 기대 효과

evaluation consumer는 duplicate prompt가 있어도 별도 key reconciliation 없이 input과 result를 zip할 수
있습니다. monitoring에서는 request 수, 실제 activation 수, block 수가 구분돼 no-match가 많은 batch를
잘못된 activation으로 해석하지 않게 됩니다.

팀은 collection 선택이 성능 구현 세부사항이 아니라 public cardinality contract가 될 수 있음을 공유하게
됩니다. 이후 batch API 리뷰에서는 duplicate input, positional alignment, summary metric의 정의를 공통
체크 항목으로 다룰 수 있으며, cache는 결과 표현과 별도의 내부 최적화로 검토하게 됩니다.

## 5. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 5ae605a | batch trigger Value Object와 API | 전체 126건 통과 |
| 리뷰 명세 | c1405d9 | duplicate order·triggered count test | 초기 구현에서 2 failures 확인 |
| 리뷰 반영 | 6b169f2 | positional result와 count semantics | runtime 15건, 전체 128건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
