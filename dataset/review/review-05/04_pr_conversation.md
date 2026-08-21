# R-A5 PR 대화 — support retrieval trace 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: `review/review-05-support-retrieval-trace` (`403cf27`) · 최종 head: `6536ec4`

## 스레드 1 — threshold를 confidence로 기록한 semantic trace

**위치** `brainwash/decomposition.py` (`SupportKnowledgeStore.retrieve_with_trace`, 초기 PR 713-719행)
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

semantic match가 성공하면 trace score에 항상 `0.6`을 넣습니다. 하지만 `0.6`은 match를 수락하는
최저 기준이고, 실제 matcher similarity가 아닙니다. 예를 들어 matcher가 `0.93`을 반환해도 현재
trace는 아래처럼 결과 강도를 잃습니다.

```python
# acceptance rule
if similarity >= 0.6:
    answer = candidate_answer

# 현재 trace: 실제 similarity와 무관하게 0.6
SupportRetrieval(..., score=0.6)
```

이 trace는 운영 분석과 evaluation에서 confidence를 비교할 근거가 되므로, `_semantic_retrieve()`가
answer와 best similarity를 함께 반환하고 trace에는 그 observed score를 기록해 주세요. 고정 score
matcher의 `0.93`이 그대로 남는 regression test도 부탁드립니다.

[scikit-learn의 decision threshold 문서](https://scikit-learn.org/stable/modules/classification_threshold.html)는
모델이 낸 score와 그 score를 threshold로 잘라 내리는 decision을 분리합니다. 여기서도 같은 구분이
필요합니다. `0.6`은 “선택할지”의 rule이고 `0.93`은 해당 후보가 실제로 받은 similarity입니다. 둘을
같은 trace field에 기록하면 threshold를 바꾼 뒤 과거 관측값까지 달라 보이게 됩니다.

**작업자 · 답변 :speech_balloon:**

맞습니다. threshold와 관측값을 같은 field에 섞었습니다. `_semantic_retrieve()`의 반환을
`(answer, score)`로 바꾸고, semantic trace는 matcher가 실제로 낸 score를 사용하겠습니다.
고정 matcher와 기존 `retrieve()` 호환을 함께 확인하는 test를 추가하겠습니다.

**리뷰어 · 후속 :mag:**

좋습니다. trace에서는 `0.6`이 “통과선”이 아니라 “확신도”처럼 읽히기 때문에 둘을 분리해야 합니다.
아래 형태가 되면 판단 기준과 관측값이 함께 보존됩니다.

```python
answer, score = self._semantic_retrieve(question, matcher)
return SupportRetrieval.for_query(question, answer, "semantic", score, ...)
```

**작업자 · 반영 :+1:**

`6536ec4 fix(review-05): preserve support retrieval confidence`에서 semantic helper가
`(answer, score)`를 반환하게 하고, trace에는 실제 best score를 넣었습니다. `0.93` 고정 matcher
test와 기존 `retrieve()` 결과 확인을 추가했습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. threshold는 candidate 선택에만 쓰이고 trace score는 실제 matcher 결과를 보존합니다.

## 스레드 2 — 새 trace 계약을 보호하는 regression test

**위치** `brainwash/decomposition.py` (`SupportRetrieval`, 초기 PR 117-144행)
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :test_tube:**

`SupportRetrieval`은 source·score·reason을 외부에 새로 공개하는 Value Object인데, 최초 PR에는 이
계약을 직접 고정하는 test가 없습니다. semantic trace가 실제 score를 보존하는지와 기존 public API가
같은 answer를 돌려주는지를 한 test에서 확인해 주세요.

또 batch API는 input마다 trace 하나를 반환한다고 설명하므로, exact match와 token이 부족한 no-match를
같이 넣어 순서·normalized query·reason을 확인하는 test가 있으면 좋겠습니다.

**작업자 · 답변 :memo:**

동의합니다. 고정 score matcher로 semantic source, `0.93`, answer와 `retrieve()`의 answer를 함께
assert하겠습니다. batch는 exact와 `"zz"`를 넣어 2개 trace의 순서, normalized query, no-match reason을
확인하겠습니다.

**리뷰어 · 후속 :bulb:**

그 범위면 구현 detail이 아니라 caller가 의존할 결과 계약을 검증합니다. 특히 batch가 no-match를
drop하지 않는지까지 확인돼 evaluation에서 입력과 결과를 zip할 수 있습니다.

**작업자 · 반영**

`502ddbc test(review-05): specify support retrieval traces`에 두 regression test를 추가했습니다.
초기 구현에서는 semantic score assertion이 실패했고, `6536ec4` 반영 뒤 decomposition test 24건이
통과했습니다.

**리뷰어 · 확인 :white_check_mark:**

source·observed score·기존 API 호환·batch alignment가 모두 test로 고정됐습니다.

## 스레드 3 — no-match score의 의미

**위치** `brainwash/decomposition.py` (`SupportKnowledgeStore.retrieve_with_trace`, 초기 PR 745-752행)
**심각도** `question`

**리뷰어 · 질문 :eyes:**

lexical no-match에는 threshold 아래의 best score를 남기고, token이 부족한 no-match에는 `None`을
남깁니다. `source == "none"`일 때 score는 “선택된 답의 confidence”가 아니라 “가장 가까웠던 lexical
candidate”로 읽어야 하는데, 이 차이를 의도한 건가요?

**작업자 · 답변 :speech_balloon:**

의도했습니다. token이 부족하면 비교 자체를 하지 않아 score가 없고, lexical 비교를 한 뒤 threshold를
못 넘으면 best score를 diagnostic signal로 남깁니다. caller는 source와 reason을 먼저 보고 score를
해석해야 합니다.

**리뷰어 · 후속 :+1:**

좋습니다. no-match를 모두 `None`으로 평탄화하지 않아 tuning 시 근거가 남습니다. 이번 PR에서는
reason으로 경로를 구분하니 추가 변경 요청은 없습니다.

## 스레드 4 — 기존 `retrieve()` caller 호환성

**위치** `brainwash/decomposition.py` (`SupportKnowledgeStore.retrieve`, 초기 PR 703-704행)
**심각도** `question`

**리뷰어 · 질문 :speech_balloon:**

기존 `retrieve()`가 trace API에 위임합니다. 기존 caller가 answer만 받는 계약은 그대로인데,
trace 생성이 예외 처리나 answer selection 순서를 바꾸지는 않는지 확인한 근거가 있나요?

**작업자 · 답변 :memo:**

exact → semantic → lexical 순서와 answer-type gating은 기존 helper를 그대로 사용했습니다.
`retrieve()`는 `retrieve_with_trace(question).answer`만 반환하므로 반환형도 `str | None`으로 유지됩니다.
새 semantic regression test에서 trace answer와 `retrieve()` answer가 모두 `"Seoul"`인지 확인했습니다.

**리뷰어 · 확인 :white_check_mark:**

기존 API를 별도 구현으로 복제하지 않고 trace의 answer를 단일 출처로 쓴 점도 좋습니다. 테스트로
호환 경계가 확인됐습니다.

## 스레드 5 — batch trace의 입력 정렬

**위치** `brainwash/decomposition.py` (`SupportKnowledgeStore.retrieve_many_with_trace`, 초기 PR 757-761행)
**심각도** `question`

**리뷰어 · 질문 :thinking:**

batch helper가 generator도 받을 수 있는데, no-match를 제외하거나 sorting하지 않고 입력 순서대로 한
trace씩 반환하는 계약인가요? evaluation 쪽에서 input과 trace를 zip하려면 이 부분이 중요합니다.

**작업자 · 답변 :speech_balloon:**

네. list comprehension으로 iterable을 한 번 순회하고 모든 question에 대해 trace 하나를 반환합니다.
no-match도 `source="none"` trace로 남기며 reordering은 하지 않습니다. batch test에서 exact와 no-match
두 trace의 순서를 확인했습니다.

**리뷰어 · 확인 :tada:**

입력 cardinality와 순서가 보존돼 evaluation consumer가 별도 reconciliation 없이 사용할 수 있습니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

semantic threshold와 observed similarity를 분리해 trace의 정보 정확성을 보완했고, 기존 answer API와
batch 입력 정렬도 regression test로 고정했습니다. focused decomposition test 24건과 전체 test 101건
통과를 확인하여 승인합니다.
