# R-A7 PR 대화 — promotion candidate 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: review/review-07-promotion-candidates (5be9d0c) · 최종 head: 9aae539

## 스레드 1 — aggregate 결과의 record ID 순서가 보장되지 않음

**위치** brainwash/memory/update_db.py (promotion_candidates, 초기 PR 448-449행)  
**심각도** blocking · **Change Request**

**리뷰어 · 댓글 :thinking:**

GROUP_CONCAT(id)를 그대로 API의 record_ids로 내보내고 있습니다. candidate는 audit와 training 입력의 근거가 될 수 있는데, 이 순서가 execution마다 달라지면 같은 cluster도 다른 JSON처럼 보입니다.

[SQLite aggregate 함수 문서](https://www.sqlite.org/lang_aggfunc.html)는 aggregate에 ORDER BY가 없으면 입력 순서가 임의이고 invocation마다 바뀔 수 있다고 설명합니다. group_concat()도 같은 규칙입니다.

    -- 현재: concatenated ID order가 정해지지 않음
    GROUP_CONCAT(id)

    -- 필요한 계약: id 순서가 고정된 candidate
    ORDER BY subject_key, relation, scope, id

정렬된 row를 기준으로 record_ids를 만들고, insert 순서와 관계없이 (memory-a, memory-b)가 나오는 test를 추가해 주세요. :mag:

**작업자 · 답변 :speech_balloon:**

맞습니다. 바깥 query의 ORDER BY가 group 내부의 GROUP_CONCAT 순서를 보장한다고 생각했는데 별개였습니다. candidate 자체가 downstream diff 입력이 될 수 있으니 정렬을 명시해야 합니다. 개별 row를 key별로 모으는 방식으로 바꾸고 ID order를 test로 고정하겠습니다.

**리뷰어 · 후속 :+1:**

좋습니다. SQLite version별 aggregate syntax에 의존하지 않고 Python에서 이미 정렬된 rows를 모으면 target canonicalization도 같은 순회에서 처리할 수 있겠습니다. query 결과 순서와 output 순서가 한 곳에서 보이는 장점도 있습니다.

**작업자 · 반영 :white_check_mark:**

9aae539 fix(review-07): stabilize promotion candidates에서 active rows를 key와 ID로 정렬해 읽고, grouping 후 record_ids를 그 순서로 구성했습니다. memory-b를 먼저 insert해도 output은 (memory-a, memory-b)인지 검증하는 regression test를 추가했습니다.

**리뷰어 · 확인 :tada:**

확인했습니다. candidate의 identity와 출력 순서가 분리되지 않고 같은 정렬 기준으로 고정됐습니다.

## 스레드 2 — candidate target 비교가 promotion policy와 다름

**위치** brainwash/memory/update_db.py (GROUP_CONCAT(DISTINCT target), 초기 PR 449행)  
**심각도** important · **Change Request**

**리뷰어 · 댓글 :eyes:**

기존 PromotionPolicy의 ConsistentTargetRule은 공백을 정리하고 casefold한 target으로 충돌을 판정합니다. 그런데 candidate query는 raw target으로 DISTINCT를 적용합니다. 따라서 아래 두 record는 policy에서는 같은 target인데, candidate API에서는 두 target이 되어 manual review로 갑니다.

    "Seoul"     # normalized: "seoul"
    " seoul "   # normalized: "seoul"

후보 조회가 promotion 전 검토용이면 policy보다 다른 안전 기준을 만들기보다 같은 정규화 계약을 재사용하는 편이 좋겠습니다. 같은 cluster가 target 하나이고 ready 상태로 나오는 test도 부탁드립니다. :test_tube:

**작업자 · 답변 :memo:**

동의합니다. raw target을 보여 주고 싶다는 이유로 비교 기준까지 raw로 둔 것이 문제였습니다. 비교는 _normalize_key()로 하고, 출력은 정렬된 record 중 먼저 나온 표시용 target을 trim해서 유지하겠습니다.

**리뷰어 · 후속 :bulb:**

그 경계면 충분합니다. policy decision은 normalized value에만 의존하고, 사람이 보는 label은 원래 spelling을 유지할 수 있습니다. 같은 normalized key가 여러 번 나와도 첫 canonical label만 남기는지 확인하면 됩니다.

**작업자 · 반영 :+1:**

같은 9aae539에서 targets_by_key를 normalized target으로 만들었습니다. "Seoul"과 " seoul "을 넣은 cluster가 targets == ("Seoul",)이고 ready_for_training인지를 test로 고정했습니다.

**리뷰어 · 확인 :white_check_mark:**

candidate API와 PromotionPolicy가 같은 conflict 의미를 사용하게 됐습니다. display label과 판단 key도 분리되어 있습니다.

## 스레드 3 — threshold 0의 처리

**위치** brainwash/memory/update_db.py (promotion_candidates)  
**심각도** question

**리뷰어 · 질문 :thinking:**

many_min=0이면 모든 active group이 후보가 될 수 있습니다. caller 오류를 조용히 허용하지 않고 ValueError로 처리한 것은 기존 MinimumRecordsRule과 맞추려는 의도인가요?

**작업자 · 답변 :speech_balloon:**

네. promotion policy도 threshold를 1 이상으로 제한합니다. 조회 API만 0을 허용하면 evidence가 없는 candidate라는 별도 의미가 생겨 policy와 어긋납니다. 그래서 같은 제약을 적용했습니다.

**리뷰어 · 확인 :+1:**

read model도 policy가 허용하는 threshold domain을 그대로 쓰는 점을 확인했습니다. 추가 변경은 없습니다.

## 스레드 4 — high-risk candidate의 의미

**위치** PromotionCandidate.requires_manual_review  
**심각도** question

**리뷰어 · 질문 :shield:**

high-risk record가 하나라도 있으면 candidate 전체가 requires_manual_review입니다. 이 값이 candidate를 숨김이 아니라 자동 promotion을 하지 않음으로 읽혀야 하는데, caller가 후보 목록에서 제외하지는 않는 계약인가요?

**작업자 · 답변 :memo:**

맞습니다. 이 API는 검토 대상을 찾는 read model이라 high-risk cluster도 반환합니다. ready_for_training=False와 requires_manual_review=True로만 표시하고, 실제 승인 또는 제외는 호출자가 수행합니다.

**리뷰어 · 확인 :white_check_mark:**

고위험 후보를 숨기지 않아 review queue에서 누락되지 않고, 자동 promotion 경계도 유지됩니다.

## 스레드 5 — lifecycle audit와 persisted status의 책임

**위치** promotion_candidates의 WHERE status = active  
**심각도** question

**리뷰어 · 질문 :clock1:**

lifecycle event table에는 inactive 평가가 남을 수 있는데 candidate query는 memory_records.status만 봅니다. lifecycle audit를 candidate selection의 source of truth로 쓰지 않는 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

lifecycle event는 특정 시점의 audit 기록이고, persisted record status를 변경하지 않습니다. 후보 조회는 현재 DB가 서비스할 active memory snapshot을 기준으로 하고, audit event는 왜 그 상태가 평가됐는지 추적하는 용도입니다. event 하나로 현재 record를 implicit하게 비활성화하면 lifecycle import 순서가 selection 결과를 바꾸게 됩니다.

**리뷰어 · 확인 :+1:**

현재 상태와 audit history를 분리한 책임 경계가 명확합니다. 이 PR에서는 status=active만 보는 것이 맞겠습니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

candidate 결과의 record ID 순서를 결정적으로 만들고, target conflict 판정을 기존 promotion policy와 일치시켰습니다. threshold, high-risk review, lifecycle audit의 책임도 확인했습니다. tests.test_update_db 12건과 전체 test 124건 통과를 확인하여 승인합니다.
