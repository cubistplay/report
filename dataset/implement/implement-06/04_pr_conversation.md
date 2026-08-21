# I-A6 PR 대화 — memory promotion Policy 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `39dbc7a` · Change Request 없음

## 스레드 1 — ledger와 promotion 판단의 책임 경계

**위치** `brainwash/memory/promotion.py` (`PromotionPolicy`), `brainwash/memory/ledger.py`

**리뷰어 · 질문 🔎**

기존 `promotion_report()`의 grouping을 별도 Policy로 옮겼습니다. ledger 안에 helper를 두는 대신
새 module로 분리한 이유가 있나요?

**작업자 · 답변 💬**

ledger는 record를 만들고 active snapshot·artifact를 관리하는 persistence 경계입니다. 반면
minimum count, target consistency, high-risk는 training workflow가 바뀌면 달라질 선택 정책입니다.
그래서 `PromotionPolicy`가 Decision을 만들고 ledger는 active record만 전달하게 분리했습니다.

**리뷰어 · 후속 질문 💭**

그러면 신규 gate가 필요할 때 ledger 수정 없이 `PromotionRule`만 추가할 수 있겠네요. report의
serialization도 Policy 쪽에 둔 이유가 같은 건가요?

**작업자 · 답변**

네. report는 eligibility 판단의 설명 결과라 `PromotionDecision.to_dict()`에서 함께 관리합니다.
ledger에는 file write만 남겨 규칙 추가가 schema export 흐름까지 번지지 않게 했습니다.

**리뷰어 · 확인 ✅**

persistence와 training eligibility가 분리돼 이후 promotion gate 확장 범위가 작아졌습니다.

## 스레드 2 — conflict와 high-risk의 기본 안전 정책

**위치** `ConsistentTargetRule`, `HighRiskReviewRule`

**리뷰어 · 질문 🔎**

target이 둘인 cluster와 high-risk cluster를 모두 `ready_for_training=False`로 둡니다. 둘 다 같은
차단으로 보이지만 처리 방식은 다른데 구분이 충분한가요?

**작업자 · 답변 💬**

둘 다 automatic promotion은 막지만 `requires_manual_review`와 reason으로 구분합니다.
conflict는 `conflicting_targets: 2 active targets`를 남겨 source correction을 정리해야 하고,
high-risk는 target이 일관돼도 승인 workflow가 필요하다는 뜻으로 `high_risk_requires_review`를
남깁니다.

**리뷰어 · 후속 질문 💭**

high-risk override가 default 동작을 우회하는 hidden flag가 되지는 않나요?

**작업자 · 답변**

기본 API는 항상 risk review를 요구합니다. override는 `PromotionPolicy(many_min=..., 
require_risk_review=False)`를 caller가 명시적으로 구성해야 해서, report와 호출부에서 reviewed
workflow 선택을 확인할 수 있습니다. conflict gate는 override하지 않습니다.

**리뷰어 · 확인 👍**

안전한 기본값을 유지하면서 예외 workflow도 명시적으로 표현돼 의도가 분명합니다.

## 스레드 3 — 기존 호출 형식과 deterministic report

**위치** `MemoryLedger.promotion_report`

**리뷰어 · 질문 🔎**

기존에는 `promotion_report(many_min)`만 호출했습니다. Policy를 받게 바꾸면서 existing caller의
결과 shape나 ordering이 바뀌지 않는지 확인했나요?

**작업자 · 답변 💬**

positional `many_min`은 그대로 default Policy 생성에 사용합니다. 기존 key, size,
`ready_for_training`, record ID는 유지했고 target·reason·manual review 필드만 추가했습니다.
cluster는 size 내림차순 뒤 key로 정렬해 동률에서도 deterministic하게 만들었습니다.

**리뷰어 · 후속 질문 💭**

Update DB는 ledger artifact를 ingest합니다. 새 report 필드가 그 경로에 영향을 주지 않는다는
검증도 있나요?

**작업자 · 답변**

promotion report는 ledger artifact 중 하나이고 Update DB가 ingest하는 memory ledger JSONL의 record
shape는 바꾸지 않았습니다. `tests.test_update_db` 4건과 전체 suite 106건을 함께 통과시켜 해당
ingestion 경로를 확인했습니다.

**리뷰어 · 확인 📌**

기존 convenience API, memory ledger JSONL, Update DB ingestion을 건드리지 않고 policy 결과만
확장한 범위가 명확합니다.

## 승인

**리뷰어 · Approve ✅**

promotion eligibility를 Policy·Rule Strategy로 분리했고, conflict/high-risk 안전 경계와
기존 ledger/Update DB 계약이 테스트로 확인됐습니다. 승인합니다.
