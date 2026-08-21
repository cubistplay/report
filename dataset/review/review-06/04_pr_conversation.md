# R-A6 PR 대화 — lifecycle audit import 검토

> Synthetic GitHub artifact: true
> 최초 검토 branch: `review/review-06-lifecycle-audit` (`b61a2d0`) · 최종 head: `8e32f99`

## 스레드 1 — 문자열 `"false"`가 active event가 되는 문제

**위치** `brainwash/memory/update_db.py` (`ingest_memory_lifecycle_rows`, 초기 PR 363행)
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

`active`는 JSON artifact에서 오는 값인데 지금은 Python truthiness로 0/1을 정합니다. 그래서 producer가
`"false"`를 보낼 경우 active=1로 저장됩니다.

```python
bool("false")  # True
```

[Python Truth Value Testing 문서](https://docs.python.org/3/library/stdtypes.html#truth-value-testing)는 빈 문자열만 false로
분류하고, 비어 있지 않은 문자열은 true로 평가된다고 설명합니다. 즉 `if row.get("active")`는 JSON boolean을
검증하는 코드가 아닙니다.

`bool`, `0/1`, `"true"/"false"`만 명시적으로 허용하고, 그 밖의 값은 `ValueError`로 끊어 주세요. `"false"`가
0으로 저장되고 `"still deciding"`이 실패하는 regression test도 같이 부탁드립니다. :mag:

**작업자 · 답변 :speech_balloon:**

맞습니다. producer가 JSON encoder를 쓰면 bool이 오겠지만, 수동으로 만든 artifact나 이전 도구는 문자열을
낼 수 있습니다. truthiness에 기대지 않고 `_coerce_lifecycle_active()`를 두겠습니다. 허용값 밖의 문자열을
조용히 false로 처리하지 않고 오류로 남기겠습니다.

**리뷰어 · 후속 :+1:**

좋습니다. `"false"`를 false로 바꾸는 것만으로 끝내면 오타도 false가 될 수 있어서, 허용값을 좁히는 편이
import 실패 원인을 찾기 쉽습니다. 숫자는 SQLite 입력과 맞춰 `0`, `1`만 받아도 충분해 보입니다.

**작업자 · 반영 :white_check_mark:**

`8e32f99 fix(review-06): normalize lifecycle audit imports`에서 boolean, 정수 `0/1`, 대소문자와 공백을
정리한 `"true"/"false"`만 받도록 했습니다. `"false"` artifact가 0으로 저장되는 test와 알 수 없는 문자열이 `ValueError`가 되는 test를 추가했습니다.

**리뷰어 · 확인 :tada:**

확인했습니다. Python의 일반 truthiness와 artifact schema의 boolean 계약이 분리됐고, 잘못된 입력도 숨기지 않습니다.

## 스레드 2 — lifecycle import 결과가 report에 남지 않음

**위치** `brainwash/memory/update_db.py` (`ingest_run_artifacts`, 초기 PR 320-322행)
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :eyes:**

`memory_lifecycle.jsonl`을 읽어도 반환값을 버립니다. run import를 호출한 쪽은 correction, memory, eval,
training 개수는 report에서 확인할 수 있는데 lifecycle만 몇 건 반영됐는지 알 수 없습니다.

artifact가 존재하지만 모든 row가 비어 있어 skip된 경우와 정상 import된 경우가 같은 report로 보입니다. 기존
`UpdateDbIngestReport`에 `lifecycle_events`를 추가하고 `to_dict()`에도 넣어 주세요. run directory를 통한
import test에서 count와 저장 event를 같이 확인하면 좋겠습니다. :test_tube:

**작업자 · 답변 :memo:**

동의합니다. 현재 API가 다른 artifact count를 이미 report하므로 lifecycle만 제외할 이유가 없습니다.
`lifecycle_count`를 지역에서 보관해 report field로 넘기고, JSON 출력에도 같은 키를 넣겠습니다.

**리뷰어 · 후속 :bulb:**

네, `ingest_memory_lifecycle_rows()`의 반환값을 그대로 쓰면 새 counting 규칙을 만들 필요도 없습니다. artifact가
없는 run은 기본값 0을 유지하면 기존 caller도 깨지지 않겠습니다.

**작업자 · 반영 :+1:**

같은 `8e32f99`에서 `lifecycle_events: int = 0`을 report에 추가했습니다. run artifact test는 report가 1이고,
`lifecycle_events(memory_id)`의 첫 event가 inactive인지 함께 확인합니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. import 결과와 저장 상태를 둘 다 검증하므로 운영자가 report만 보고도 lifecycle artifact 처리량을 확인할 수 있습니다.

## 스레드 3 — ledger와 event의 foreign key 순서

**위치** `brainwash/memory/update_db.py` (`UpdateDb.ingest_run_artifacts`, 초기 PR 287-336행)
**심각도** `question`

**리뷰어 · 질문 :thinking:**

lifecycle table이 `memory_records`를 참조합니다. 같은 run에 ledger와 lifecycle artifact가 모두 있으면 항상 ledger를
먼저 import한다는 전제인가요? 순서가 반대면 foreign key error가 날 수 있어 확인하고 싶습니다.

**작업자 · 답변 :speech_balloon:**

네. `ingest_run_artifacts()`에서 `memory_ledger.jsonl` block이 lifecycle block보다 앞에 있고, 둘 다 있을 때
memory record를 먼저 upsert합니다. lifecycle audit는 독립 생성물이 아니라 해당 ledger record의 평가 이력이므로,
record 없이 event만 있는 artifact는 데이터 불완전으로 보고 DB 제약이 실패하게 두었습니다.

**리뷰어 · 확인 :+1:**

호출 순서와 foreign key가 같은 계약을 표현하고 있네요. 누락 record를 orphan event로 저장하지 않는 판단도 맞습니다. 변경 요청은 없습니다.

## 스레드 4 — 같은 artifact 재실행의 의미

**위치** `brainwash/memory/schema.sql` (`memory_lifecycle_events`, 초기 PR 70-80행)
**심각도** `question`

**리뷰어 · 질문 :repeat:**

`INSERT OR REPLACE`와 `(memory_id, evaluated_at, source)` unique key 조합이면 같은 run을 다시 import해도 event는
늘지 않습니다. audit import를 재실행 가능하게 하려는 의도인가요? reason이 바뀐 artifact도 같은 event를 갱신합니다.

**작업자 · 답변 :memo:**

의도한 동작입니다. `evaluated_at`과 source가 같은 row는 동일 평가의 재전송으로 보고 마지막 artifact 내용을
반영합니다. 평가를 다시 수행했다면 새 `evaluated_at`을 생성해야 합니다. run을 재시도할 때 event가 중복되는 것보다
동일 관측을 교체하는 편이 consumer가 단순합니다.

**리뷰어 · 확인 :white_check_mark:**

unique key가 재실행 단위와 맞고, timestamp가 새 평가의 identity라는 설명도 일관됩니다. 여기서는 추가 변경이 필요 없습니다.

## 스레드 5 — 최신 event 정렬의 입력 형식

**위치** `brainwash/memory/update_db.py` (`UpdateDb.lifecycle_events`, 초기 PR 372-382행)
**심각도** `question`

**리뷰어 · 질문 :clock1:**

조회가 `evaluated_at DESC` 문자열 정렬에 의존합니다. lifecycle producer가 UTC ISO 8601 offset 형식으로 시간을 쓰는
계약인가요? 서로 다른 형식이 섞이면 최신 event가 어긋날 수 있습니다.

**작업자 · 답변 :speech_balloon:**

현재 lifecycle policy가 `datetime.isoformat()` UTC 값을 쓰고, artifact는 그 값을 그대로 전달합니다. 이 PR에서
문자열을 다시 파싱하면 기존 artifact contract를 넓히게 되므로, producer의 UTC ISO 8601 계약을 유지했습니다.

**리뷰어 · 확인 :+1:**

생성 경로가 정규화한 UTC timestamp를 쓰는 것을 확인했습니다. 현재 범위에서는 저장 계층이 별도 날짜 정책을 갖지 않는 편이 낫겠습니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

외부 artifact의 boolean 계약을 명시적으로 검증하고, lifecycle import count를 report에 노출했습니다. ledger 선행
import, 재실행 identity, timestamp 정렬의 책임도 현재 계약과 일치함을 확인했습니다. `tests.test_update_db` 10건과
전체 test 122건 통과를 확인하여 승인합니다.
