# R-A8 PR 대화 — memory snapshot export 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: review/review-08-memory-snapshots (1d6cd78) · 최종 head: fa09f20

## 스레드 1 — 한 snapshot 안에서 artifact별 as_of가 달라짐

**위치** brainwash/memory/ledger.py (write_artifacts, 초기 PR 378-398행)  
**심각도** blocking · **Change Request**

**리뷰어 · 댓글 :thinking:**

active_memory와 memory_lifecycle에는 as_of를 전달했는데, memory_conflicts, memory_index, promotion_report는 인자 없이 호출합니다. 예를 들어 record가 snapshot 시점에는 아직 valid_from 전이고 현재는 active면, active_memory는 비어 있는데 index와 promotion report에는 그 record가 들어갑니다.

    snapshot as_of: 2026-08-20
    record valid_from: 2026-08-21

    active_memory.jsonl       -> []
    memory_index.json         -> record 포함   # 현재 구현

같은 export directory의 파일이 서로 다른 state를 나타내므로 snapshot manifest의 as_of도 신뢰하기 어렵습니다. conflicts, exact_index, promotion_report에도 선택적 as_of를 전달하고, 과거 snapshot에서 세 artifact가 모두 비어 있는 regression test를 추가해 주세요. :mag:

**작업자 · 답변 :speech_balloon:**

맞습니다. active view와 lifecycle만 바꾼 뒤 기존 helper 호출을 놓쳤습니다. 각 helper가 active_records의 as_of를 받도록 확장하고 write_artifacts에서 validated value 하나를 모든 호출에 전달하겠습니다.

**리뷰어 · 후속 :+1:**

좋습니다. write_artifacts가 각 helper에 다른 datetime 객체를 만들지 않고 snapshot_as_of 하나를 전달하면, bundle의 경계가 코드에서도 바로 보입니다. index뿐 아니라 promotion report의 cluster도 같이 비어 있는지 확인하면 충분합니다.

**작업자 · 반영 :white_check_mark:**

fa09f20 fix(review-08): align snapshot artifact boundaries에서 conflicts, exact_index, promotion_report에 as_of를 추가했습니다. snapshot 당시 inactive이고 현재 active인 record로 active memory, index, promotion clusters가 모두 비어 있는 test를 추가했습니다.

**리뷰어 · 확인 :tada:**

확인했습니다. 동일한 validated as_of가 모든 derived view에 전달돼 bundle 내부의 시간 경계가 일치합니다.

## 스레드 2 — naive datetime을 시스템 timezone으로 해석함

**위치** brainwash/memory/ledger.py (write_artifacts, 초기 PR 378행)  
**심각도** important · **Change Request**

**리뷰어 · 댓글 :clock1:**

as_of가 naive datetime이면 MemoryLifecyclePolicy가 astimezone(UTC)로 변환하는 과정에서 실행 환경의 local timezone 의미가 섞일 수 있습니다. snapshot은 재현 가능한 audit artifact라 timezone이 없는 시각을 허용하면 안 됩니다.

[Python datetime 문서](https://docs.python.org/3/library/datetime.html#aware-and-naive-objects)는 naive datetime이 UTC인지 local time인지 다른 timezone인지가 프로그램의 해석에 달렸다고 설명합니다. 반면 aware datetime은 특정 시점을 표현합니다.

    datetime(2026, 8, 21, 9, 0)                 # UTC인지 알 수 없음
    datetime(2026, 8, 21, 9, 0, tzinfo=UTC)     # 특정 시점

write_artifacts의 as_of가 timezone-aware가 아니면 ValueError로 실패하도록 하고, naive input regression test를 추가해 주세요. :test_tube:

**작업자 · 답변 :speech_balloon:**

동의합니다. 내부 lifecycle helper는 기존 호환성 때문에 그대로 두되, snapshot export라는 public artifact boundary에서는 aware datetime만 허용하겠습니다. tzinfo가 있더라도 utcoffset이 None인 경우도 걸러내겠습니다.

**리뷰어 · 후속 :bulb:**

그 범위가 적절합니다. 기존 policy API의 해석을 넓게 바꾸지 않으면서, 재현 가능한 export 경로의 입력 계약만 명확히 할 수 있습니다.

**작업자 · 반영 :+1:**

같은 fa09f20에서 _validated_snapshot_as_of()를 추가했습니다. tzinfo 또는 utcoffset이 없는 값은 snapshot as_of must be timezone-aware ValueError로 거절합니다. naive datetime export test가 통과합니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. 시스템 timezone에 의존하던 해석이 artifact API 경계에서 제거됐습니다.

## 스레드 3 — 원본 ledger가 snapshot filter를 받지 않는 이유

**위치** write_artifacts의 memory_ledger.jsonl  
**심각도** question

**리뷰어 · 질문 :eyes:**

active_memory는 snapshot view인데 memory_ledger는 모든 record를 그대로 export합니다. 이 파일도 as_of로 필터해야 하는 건 아닌가요?

**작업자 · 답변 :memo:**

원본 ledger는 snapshot view의 입력과 audit 재현용 원장입니다. filter하면 snapshot 당시 inactive였던 record의 valid_from, valid_until, status를 사후에 설명할 수 없습니다. 대신 snapshot manifest와 lifecycle artifact가 어떤 view를 선택했는지 표현합니다.

**리뷰어 · 확인 :+1:**

원장과 derived snapshot의 책임이 분리되어 있네요. 원본을 유지하는 편이 이후 재평가에도 적절합니다.

## 스레드 4 — 빈 ledger의 snapshot manifest

**위치** snapshot_manifest  
**심각도** question

**리뷰어 · 질문 :thinking:**

record가 하나도 없으면 decisions에서 as_of를 얻을 수 없습니다. 빈 ledger도 caller가 지정한 as_of를 manifest에 기록하나요?

**작업자 · 답변 :speech_balloon:**

네. decisions가 비어 있으면 _resolved_lifecycle_policy(as_of)의 as_of_iso를 사용합니다. 그래서 empty export도 active/inactive 0과 호출자가 지정한 snapshot boundary를 남깁니다.

**리뷰어 · 확인 :white_check_mark:**

빈 artifact도 경계 정보가 사라지지 않아 consumer가 special case를 만들 필요가 없습니다.

## 스레드 5 — conflict artifact의 역할

**위치** memory_conflicts.json  
**심각도** question

**리뷰어 · 질문 :shield:**

conflict가 있는 cluster는 active snapshot이 priority로 하나를 선택해도, conflict artifact에는 모든 active record가 남습니다. 이 차이는 의도한 건가요?

**작업자 · 답변 :memo:**

의도했습니다. active snapshot은 retrieval에 사용할 winner view이고, conflict artifact는 promotion 전에 사람이 확인해야 할 competing evidence입니다. 따라서 둘이 같은 cardinality를 가질 필요는 없습니다. 다만 둘 다 같은 as_of를 써야 한다는 첫 번째 Change Request는 맞습니다.

**리뷰어 · 확인 :+1:**

winner selection과 conflict audit의 목적이 다르다는 점이 명확합니다. time boundary만 맞추면 현재 구조를 유지해도 되겠습니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

snapshot export의 모든 derived artifact가 하나의 as_of를 공유하고, timezone-aware input만 받도록 보완됐습니다. 원본 ledger와 snapshot view의 책임도 명확합니다. tests.test_memory_ledger 12건과 전체 test 126건 통과를 확인하여 승인합니다.
