# I-A9 PR 대화 — memory match Strategy 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `1fe0233` · Change Request 없음

## 스레드 1 — match evidence와 final safety policy

**위치** `brainwash/algorithms/memory_edit.py` (`MemoryMatchStrategy`, `MemoryEditStore`)

**리뷰어 · 질문 🔎**

match Strategy가 score를 반환하지만 threshold와 answer-type block은 `MemoryEditStore`에 남겼습니다.
Strategy가 trigger acceptance까지 맡지 않는 이유가 있나요?

**작업자 · 답변 💬**

exact/paraphrase와 token overlap은 candidate evidence를 만드는 방식만 다릅니다. threshold와 answer-type은
어느 candidate든 같은 memory safety contract여야 합니다. Strategy마다 acceptance를 넣으면 low score나
wrong answer type의 block reason이 route별로 달라질 수 있어 store에 공통으로 남겼습니다.

**리뷰어 · 후속 질문 💭**

그러면 새 semantic match를 추가해도 strict fallback과 verifier는 별도 분기 없이 그대로 적용되겠네요.

**작업자 · 답변**

네. 새 Strategy는 `MemoryMatchCandidate`만 반환하면 되고, `MemoryMatchPolicy`와 store trigger path가
기존처럼 threshold·answer-type·verifier를 처리합니다. candidate route 확장과 memory safety를 분리한
이유입니다.

**리뷰어 · 확인 ✅**

candidate 탐색의 확장성과 최종 correction safety가 분리돼 책임 경계가 명확합니다.

## 스레드 2 — exact 우선순위와 subject-aware overlap

**위치** `ExactMemoryMatchStrategy`, `SubjectTokenOverlapStrategy`

**리뷰어 · 질문 🔎**

exact/paraphrase를 overlap보다 먼저 시도하고, overlap에는 subject token guard를 적용합니다. 기존
exact 질문이 subject metadata와 다를 때도 trigger되는 동작은 보존되나요?

**작업자 · 답변 💬**

보존됩니다. exact Strategy는 subject guard보다 먼저 exact candidate를 반환합니다. overlap Strategy만
subject가 있는 edit에서 query에 subject token이 없으면 defer합니다. 따라서 canonical prompt나
paraphrase의 explicit correction은 정확히 일치할 때 우선 적용되고, loose overlap만 오답 entity로
확장되는 것을 막습니다.

**리뷰어 · 후속 질문 💭**

동점 candidate는 어떤 edit를 선택하나요?

**작업자 · 답변**

`MemoryMatchPolicy`는 score가 strictly greater일 때만 best를 교체합니다. 따라서 동점은 input edit
order의 첫 candidate를 유지해 이전 loop의 tie behavior와 동일합니다.

**리뷰어 · 확인 👍**

exact correction 우선과 subject guard의 적용 위치, tie behavior가 모두 명확하게 보존됐습니다.

## 스레드 3 — custom matcher와 explicit disable

**위치** `MemoryEditStore.__init__`, `MemoryMatchPolicy`

**리뷰어 · 질문 🔎**

`None`은 default Strategy를 만들고 빈 tuple은 triggering을 끕니다. custom Strategy가 candidate를
반환하지 않을 때 기존 verifier path가 잘못 strict fallback으로 가는 일은 없나요?

**작업자 · 답변 💬**

candidate가 없으면 store는 기존과 같은 `memory_triggered=False`, `reason=no_memory_records` trigger를
반환합니다. `answer_with_verifier()`는 edit가 없는 trigger에서 model output을 그대로 가진 `base` mode를
반환하므로 strict fallback으로 가지 않습니다. 빈 tuple과 recording Strategy test로 그 경계를 확인했습니다.

**리뷰어 · 후속 질문 💭**

custom Strategy도 raw prompt와 각 edit을 받으니 external semantic matcher를 붙일 준비가 된 셈이네요.

**작업자 · 답변**

네. injection test는 raw query와 `ms-hq` edit ID를 확인합니다. 다만 external matcher의 score calibration은
별도 정책이 필요하므로, 이번 PR은 common safety gate를 유지하는 extension boundary까지만 다룹니다.

**리뷰어 · 확인 📌**

default, explicit disable, custom extension 모두 verifier와 strict fallback의 기존 의미를 보존합니다.

## 승인

**리뷰어 · Approve ✅**

exact/paraphrase와 subject-aware overlap을 Strategy로 분리하면서 trigger safety와 verifier 흐름은
공통으로 유지했습니다. 순서·tie·empty/custom Strategy 계약이 test로 고정돼 승인합니다.
