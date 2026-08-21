# I-A2 PR 대화 — benchmark request Factory 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `559a294` · Change Request 없음

## 스레드 1 — Factory의 metadata 우선순위

**위치** `brainwash/benchmarks.py` (`BenchmarkRequestFactory.create`, 초기 PR 99-118행)

**리뷰어 · 질문 🔎**

`extra_metadata`를 먼저 복사한 뒤 `subject`, `relation`을 넣어서 Factory의 값이
최종값이 되게 했네요. adapter 고유 metadata가 이 두 값을 바꾸지 못하게 한 이유가
있나요?

**작업자 · 답변 💬**

`subject`와 `relation`은 모든 benchmark request에서 공통으로 쓰는 식별 정보입니다.
loader마다 다른 값이 들어가면 retrieval과 report 해석이 달라질 수 있어, adapter 고유
값은 `mquake_case_id`처럼 별도 key로만 추가하도록 했습니다.

**리뷰어 · 후속 질문 💭**

그렇다면 caller가 넘긴 mapping 자체는 수정하지 않아야겠네요. 재사용하는 row metadata가
있을 수 있어서요.

**작업자 · 답변**

네. `dict(spec.extra_metadata)`로 새 mapping을 만들고 그 뒤에 공통 key를 설정합니다.
테스트도 Factory 호출 뒤 원래 metadata가 그대로인지 확인합니다.

**리뷰어 · 확인 ✅**

공통 식별 정보는 Factory가 보장하고 caller 입력은 보존하는 계약이 분명합니다.

## 스레드 2 — answer type의 조회 시점과 부재 처리

**위치** `brainwash/benchmarks.py` (`BenchmarkRequestFactory.__init__`, 초기 PR 93-97행)

**리뷰어 · 질문 🔎**

Factory를 만들 때 active matcher를 미리 잡지 않고 `_infer_answer_type` 함수를
보관합니다. loader가 오래 살아 있는 경우 matcher 설정이 바뀌어도 기존 방식과 같이
동작하는지 확인하고 싶습니다.

**작업자 · 답변 💬**

`_infer_answer_type`은 request를 만들 때 `get_matcher()`를 호출합니다. Factory 생성
시점이 아니라 request 생성 시점의 matcher 설정을 따릅니다. 기존 loader도 각 row에서
같은 함수를 직접 호출했으므로 조회 시점은 유지됩니다.

**리뷰어 · 후속 질문 💭**

matcher가 없거나 answer type을 추론하지 못하면 빈 문자열 같은 값을 metadata에 넣지는
않나요?

**작업자 · 답변**

넣지 않습니다. resolver 결과가 truthy일 때만 `answer_type` key를 추가합니다. 추론하지
못한 상태를 임의의 type으로 기록하면 downstream이 실제 분류값으로 오해할 수 있어서
기존처럼 key 자체를 생략합니다.

**리뷰어 · 확인 👍**

matcher 상태를 고정하지 않고, 알 수 없는 분류를 만들어 내지도 않는 동작이 유지됩니다.

## 스레드 3 — raw schema를 Factory에 넣지 않은 이유

**위치** `brainwash/benchmarks.py` (`BenchmarkRequestSpec`, 초기 PR 78-88행)

**리뷰어 · 질문 🔎**

CounterFact의 `requested_rewrite`, RippleEdits의 `edit`처럼 입력 모양이 다른데,
Factory가 raw row를 직접 받지 않고 spec만 받는 이유가 궁금합니다.

**작업자 · 답변 💬**

raw row 해석까지 Factory가 맡으면 네 benchmark schema의 분기까지 한 클래스에 모입니다.
loader는 schema 해석을 계속 소유하고, Factory는 해석이 끝난 값의 공통 request 생성
규칙만 소유하도록 제한했습니다.

**리뷰어 · 후속 질문 💭**

그러면 MQuAKE의 multihop case나 RippleEdits의 criteria처럼 `CorrectionRequest`가 아닌
부가 결과도 Factory 밖에 남는 것이 맞겠네요.

**작업자 · 답변**

맞습니다. Factory는 fact-edit request만 만듭니다. benchmark별 평가 보조 데이터의
구조까지 공통화하면 입력 해석 경계를 다시 흐리게 하므로 기존 loader에 남겼습니다.

**리뷰어 · 확인 📌**

공통 생성 규칙만 모으고 benchmark 고유 결과는 건드리지 않은 범위가 적절합니다.

## 스레드 4 — 변환 순서와 기존 출력 보존

**위치** `brainwash/benchmarks.py` (`load_counterfact_requests`, 초기 PR 121-169행)

**리뷰어 · 질문 🔎**

Factory를 넣으면 invalid row 건너뛰기와 `limit` 적용 위치가 바뀌기 쉽습니다. 네 loader의
변환 결과가 실제로 이전과 같은지 어떤 수준으로 확인했나요?

**작업자 · 답변 💬**

각 loader의 raw row 검증과 `limit` 조건은 그대로 두고, request를 append하는 마지막
부분만 Factory 호출로 바꿨습니다. CounterFact, KnowEdit, MQuAKE, RippleEdits의 대표
입력을 변경 전 commit과 현재 commit에서 JSON으로 비교했고 차이가 없었습니다.

**리뷰어 · 후속 질문 💭**

adapter별 prompt와 locality prompt까지 비교 대상에 들어갔나요?

**작업자 · 답변**

네. JSON 전체를 비교했고, 새 테스트에서는 KnowEdit와 RippleEdits의 adapter별 prompt,
paraphrase, locality prompt도 별도로 확인했습니다.

**리뷰어 · 확인 ✅**

공통화 과정에서 순서나 adapter 고유 출력이 변하지 않았다는 근거가 충분합니다.

## 승인

**리뷰어 · Approve ✅**

공통 metadata와 answer type 규칙은 Factory로 모였고, raw schema·평가 보조 데이터·변환
순서는 각 loader에 유지됩니다. Factory 계약, adapter 테스트, 변경 전후 JSON 비교를
확인해 승인합니다.
