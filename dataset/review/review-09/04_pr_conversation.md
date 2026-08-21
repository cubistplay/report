# R-A9 PR 대화 — memory trigger batch 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: review/review-09-memory-trigger-batch (19454fa) · 최종 head: 5f56dfb

## 스레드 1 — dict result가 duplicate prompt를 지움

**위치** brainwash/algorithms/memory_edit.py (evaluate_many, 초기 PR 308-312행)  
**심각도** blocking · **Change Request**

**리뷰어 · 댓글 :thinking:**

evaluate_many가 prompt 문자열을 dict key로 사용합니다. 같은 질문이 batch에 두 번 있으면 한 trigger만 남고 request_count와 retrieve_many 결과도 줄어듭니다. evaluation에서는 input과 result를 zip할 수 있어야 하므로 이건 결과 계약이 깨지는 문제입니다.

[Python mapping types 문서](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict)는 dict가 key와 value를 연결하는 mapping이라고 설명합니다. 따라서 아래처럼 같은 key를 두 번 넣으면 value slot은 하나뿐입니다.

    {prompt: trigger for prompt in [question, question]}
    # question key 하나만 남음

result container를 prompt-keyed dict가 아니라 trigger tuple/list로 바꾸고, duplicate prompt가 있어도 input 순서와 1:1 cardinality가 보존되는 test를 추가해 주세요. :mag:

**작업자 · 답변 :speech_balloon:**

맞습니다. cache처럼 다루려다 public batch result까지 mapping으로 만들었습니다. execution cache와 returned result를 분리했어야 했습니다. batch는 input order 그대로 trigger tuple을 반환하고 retrieve_many도 같은 길이의 list를 반환하게 고치겠습니다.

**리뷰어 · 후속 :+1:**

좋습니다. 같은 prompt를 두 번 계산하는 최적화는 후속으로 별도 cache를 둘 수 있지만, public result는 input alignment가 우선입니다. trigger 안에 original prompt가 있으니 list serialization도 audit에 충분합니다.

**작업자 · 반영 :white_check_mark:**

5f56dfb fix(review-09): preserve batch trigger results에서 MemoryTriggerBatch.triggers를 tuple로 바꿨습니다. matched, unmatched, matched input이 같은 순서의 세 trigger와 세 retrieve_many result로 돌아오는 test를 추가했습니다.

**리뷰어 · 확인 :tada:**

확인했습니다. 중복과 순서가 보존되어 evaluation consumer가 별도 key reconciliation 없이 결과를 사용할 수 있습니다.

## 스레드 2 — triggered_count가 result count를 반환함

**위치** brainwash/algorithms/memory_edit.py (MemoryTriggerBatch.triggered_count, 초기 PR 159-160행)  
**심각도** important · **Change Request**

**리뷰어 · 댓글 :eyes:**

triggered_count가 len(self.triggers)입니다. no-match trigger도 batch에는 들어가므로 result 1건이 있는 unmatched batch는 triggered_count=1이 됩니다. 이 field는 name대로 memory_triggered가 true인 수를 말해야 합니다.

    batch result: [MemoryTrigger(memory_triggered=False, blocked=True)]
    expected triggered_count: 0
    current triggered_count: 1

memory_triggered를 기준으로 count하고, unmatched input 하나의 summary가 triggered_count=0, blocked_count=1인지 test로 고정해 주세요. :test_tube:

**작업자 · 답변 :memo:**

동의합니다. request_count와 triggered_count를 혼동했습니다. request_count는 전체 trigger 수를 유지하고, triggered_count는 bool signal만 합산하겠습니다. blocked_count도 같은 tuple을 기준으로 계속 계산하겠습니다.

**리뷰어 · 후속 :bulb:**

네. total, triggered, blocked는 서로 다른 operational question에 답해야 합니다. 특히 no-match가 정상적으로 많을 수 있는 evaluation에서는 triggered ratio가 핵심이라 count 의미가 정확해야 합니다.

**작업자 · 반영 :+1:**

같은 5f56dfb에서 triggered_count를 sum(trigger.memory_triggered for trigger in triggers)로 변경했습니다. no-match batch test가 triggered_count=0과 blocked_count=1을 확인합니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. 요청 수와 실제 memory activation 수가 분리되어 batch summary를 monitoring 지표로 사용할 수 있습니다.

## 스레드 3 — duplicate prompt 계산 비용

**위치** evaluate_many  
**심각도** question

**리뷰어 · 질문 :thinking:**

결과를 positional tuple로 바꾸면 duplicate prompt는 evaluate_trigger를 반복 호출합니다. 큰 evaluation batch에서 비용이 늘 수 있는데 의도한 tradeoff인가요?

**작업자 · 답변 :speech_balloon:**

현재 store는 작은 in-memory edit set이고, correctness를 위해 public API에 1:1 result를 먼저 보장했습니다. 반복 계산이 병목이 되면 내부 cache를 추가하되, cache key와 positional result는 분리하겠습니다. 이번 PR에서는 hidden cache 정책을 같이 넣지 않겠습니다.

**리뷰어 · 확인 :+1:**

현재 범위에서 observability와 alignment를 우선한 판단이 적절합니다. cache는 hit/miss와 invalidation 계약이 생기므로 별도 변경으로 보는 편이 낫겠습니다.

## 스레드 4 — reason aggregation과 positional result

**위치** MemoryTriggerBatch.reasons  
**심각도** question

**리뷰어 · 질문 :speech_balloon:**

reasons는 같은 reason을 합쳐 summary로 만듭니다. duplicate prompt도 count에 포함되어야 하는데 tuple로 바꾼 뒤 aggregation이 모든 trigger를 순회하는지 확인하고 싶습니다.

**작업자 · 답변 :memo:**

네. reasons는 positional tuple 전체를 순회하므로 동일 질문이 두 번 있으면 해당 reason도 두 번 집계됩니다. summary는 input cardinality를 압축하지 않고 count만 aggregate합니다.

**리뷰어 · 확인 :white_check_mark:**

개별 결과와 aggregated reason count의 기준 집합이 같아 monitoring 수치도 일관됩니다.

## 스레드 5 — JSON serialization 형태

**위치** MemoryTriggerBatch.to_dict  
**심각도** question

**리뷰어 · 질문 :eyes:**

triggers를 list로 직렬화하면 caller가 prompt key로 직접 찾을 수는 없지만, input alignment는 지킬 수 있습니다. batch API의 주 consumer는 key lookup보다 zip 기반 evaluation이라고 봐도 되나요?

**작업자 · 답변 :speech_balloon:**

네. 개별 prompt lookup은 기존 evaluate_trigger가 담당합니다. batch JSON은 evaluation 결과를 input 순서에 맞춰 저장하는 용도라 list가 맞습니다. 필요한 caller는 prompt와 trigger list를 같이 iterate하면 됩니다.

**리뷰어 · 확인 :+1:**

single-query lookup과 batch export의 목적이 구분되어 있습니다. list serialization으로 유지해도 되겠습니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

batch 결과가 duplicate prompt와 입력 순서를 보존하고, triggered count가 실제 memory activation만 집계하도록 보완됐습니다. summary와 positional result의 기준도 일치합니다. tests.test_memory_edit_runtime 15건과 전체 test 128건 통과를 확인하여 승인합니다.
