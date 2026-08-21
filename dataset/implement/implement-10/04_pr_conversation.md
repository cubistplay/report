# I-A10 PR 대화 — memory lifecycle Policy 통합

> Synthetic GitHub artifact: true  
> 최초 검토 head: `6fcf422` · Change Request 없음

## 스레드 1 — DB view와 in-memory lifecycle 계약

**위치** `brainwash/memory/ledger.py` (`MemoryLifecyclePolicy.evaluate`, 초기 PR 186-204행)

**리뷰어 · 질문 🔎**

DB `active_memory` view와 in-memory ledger가 이제 둘 다 status, valid_from, valid_until을
보게 됩니다. end time을 `<= as_of`에서 expired로 처리한 이유가 있나요?

**작업자 · 답변 💬**

DB view가 `valid_until > CURRENT_TIMESTAMP`를 요구하므로 end time은 exclusive입니다. 같은 instant에
record를 active로 두면 DB와 ledger snapshot이 달라집니다. Policy도 `valid_until <= as_of`를 expired로
처리해 두 경로를 맞췄습니다.

**리뷰어 · 후속 질문 💭**

invalid timestamp를 default active로 처리하지 않은 것도 data quality 문제를 숨기지 않기 위한
선택인가요?

**작업자 · 답변**

맞습니다. parse 실패는 `invalid_valid_from` 또는 `invalid_valid_until` decision으로 남기고 inactive로
처리합니다. 잘못된 validity metadata가 promotion이나 active memory export에 들어가는 것보다 audit에서
수정할 수 있게 막는 편이 안전합니다.

**리뷰어 · 확인 ✅**

DB와 ledger의 boundary semantics가 같고, bad timestamp를 숨기지 않는 기본값도 명확합니다.

## 스레드 2 — Decision Value Object와 audit artifact

**위치** `brainwash/memory/ledger.py` (`MemoryLifecycleDecision`, 초기 PR 160-173행)

**리뷰어 · 질문 🔎**

active record만 export하는 대신 lifecycle decision 전체를 JSONL로 추가했습니다. active boolean만
있어도 되는데 reason과 as_of를 함께 남긴 이유가 있나요?

**작업자 · 답변 💬**

inactive record는 future, expired, status, parse failure처럼 운영 조치가 다릅니다. `reason`이 없으면
active snapshot에서 빠진 이유를 원본 metadata까지 다시 추적해야 합니다. `as_of`도 time-dependent 결과를
재현하는 audit coordinate라 decision과 함께 보존했습니다.

**리뷰어 · 후속 질문 💭**

그러면 ledger는 Policy 결과를 직렬화할 뿐 timestamp parsing 규칙을 별도로 갖지 않겠네요.

**작업자 · 답변**

네. parsing과 active 여부는 `MemoryLifecyclePolicy`가 단일 출처이고 ledger는 decision list를
`memory_lifecycle.jsonl`로 씁니다. active snapshot, conflicts, index, promotion도 같은 active_records
경로를 사용합니다.

**리뷰어 · 확인 👍**

판정 근거와 artifact export가 분리돼 lifecycle drift를 줄이는 구조입니다.

## 스레드 3 — deterministic clock과 기존 ingestion 보존

**위치** `brainwash/memory/ledger.py` (`MemoryLedger.lifecycle_report`, 초기 PR 246-256행)

**리뷰어 · 질문 🔎**

`as_of`를 public method에 넣었습니다. runtime now를 caller가 매번 넘겨야 하는 API로 바뀌지는
않았나요?

**작업자 · 답변 💬**

기본값은 UTC now라 기존 `active_records()`와 `write_artifacts()` 호출은 그대로 동작합니다. `as_of`는
time boundary test, backfill audit, historical report처럼 explicit evaluation이 필요한 경우에만 사용합니다.
또 constructor에 Policy를 주입해 반복 evaluation의 clock도 고정할 수 있습니다.

**리뷰어 · 후속 질문 💭**

새 lifecycle artifact가 Update DB가 읽는 memory ledger JSONL schema를 건드리지 않는지도 확인했나요?

**작업자 · 답변**

기존 `memory_ledger.jsonl` row는 바꾸지 않았고 `memory_lifecycle.jsonl`은 추가 audit artifact입니다.
Update DB ingestion은 기존 ledger JSONL만 사용하므로 Update DB 8건과 전체 120건 test로 회귀가 없음을
확인했습니다.

**리뷰어 · 확인 📌**

testable clock을 추가하면서 existing operational ingestion contract는 유지됐습니다.

## 승인

**리뷰어 · Approve ✅**

active record lifecycle을 DB view와 같은 Policy로 통합했고, time-dependent exclusion reason을 audit
artifact로 남겼습니다. temporal test와 existing ledger/Update DB contract 검증을 확인하여 승인합니다.
