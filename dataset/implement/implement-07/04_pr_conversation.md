# I-A7 PR 대화 — memory retrieval Strategy 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `5b7ce2f` · Change Request 없음

## 스레드 1 — Strategy와 final selection의 경계

**위치** `brainwash/memory/update_db.py` (`RetrievalStrategy`, `retrieve_memory_edit`)

**리뷰어 · 질문 🔎**

route별 Strategy가 score까지 반환하지만 semantic rerank와 answer-type validation은 `UpdateDb`에
남겼습니다. Strategy가 candidate acceptance까지 맡지 않는 이유가 있나요?

**작업자 · 답변 💬**

route score는 SQL prefilter의 초기 신호이고, semantic rerank와 answer-type gate는 어느 route든 같은
최종 안전 정책입니다. 이를 Strategy마다 두면 score 의미와 block reason이 route별로 달라질 수 있어,
Strategy는 후보 수집까지만 제한했습니다.

**리뷰어 · 후속 질문 💭**

새 route가 threshold를 다르게 써야 하는 요구가 생기면 candidate에 threshold를 넣을 수도 있을 텐데,
지금은 확장 지점을 좁힌 셈이네요.

**작업자 · 답변**

맞습니다. 현재 threshold는 전체 Memory Store의 공통 contract입니다. route별 정책이 실제로 필요해지면
candidate가 근거 metadata를 제공하고 별도 selection Policy를 도입하는 후속 변경으로 분리하겠습니다.

**리뷰어 · 확인 ✅**

후보 조회와 최종 안전 정책의 책임이 분리돼, 현재 score 계약을 유지하면서 확장 여지도 남았습니다.

## 스레드 2 — default order와 빈 목록의 의미

**위치** `default_retrieval_strategies`, `UpdateDb.__init__`

**리뷰어 · 질문 🔎**

`None`은 default 네 route를 만들고 빈 tuple은 lookup을 disable합니다. 둘을 같은 “설정 없음”으로
해석하지 않은 이유가 있나요?

**작업자 · 답변 💬**

`None`은 caller가 정책을 지정하지 않았다는 뜻이라 established default order를 씁니다. `()`는 caller가
의도적으로 Memory Store retrieval을 끈다는 명시 설정입니다. 기존 정확한 match가 있어도 trace가
`route=none`이 되는 test로 이 차이를 고정했습니다.

**리뷰어 · 후속 질문 💭**

custom Strategy가 raw와 normalized prompt를 둘 다 받는 것도 같은 취지인가요?

**작업자 · 답변**

네. SQL key match는 normalized 값이 필요하지만 relation signal이나 future semantic backend는 원본 문장을
필요로 할 수 있습니다. constructor injection test가 공백·대문자가 있는 raw query와 normalized query를
각각 확인합니다.

**리뷰어 · 확인 👍**

default, explicit disable, custom extension의 의미가 겹치지 않고 test로 구분됐습니다.

## 스레드 3 — 기존 FTS와 audit contract 보존

**위치** `FullTextRetrievalStrategy`, `UpdateDb.log_retrieval`

**리뷰어 · 질문 🔎**

FTS table 부재 시 기존처럼 빈 candidate로 처리합니다. 클래스로 옮기면서 OperationalError 처리나
retrieval log route가 달라지지는 않았나요?

**작업자 · 답변 💬**

FTS Strategy는 기존 SQL, rank-to-score 계산, `OperationalError`를 빈 목록으로 바꾸는 정책을 그대로
옮겼습니다. candidate가 없으면 `UpdateDb`가 기존과 같은 `route=none`, `no_active_memory_match` trace를
생성하고 `log_retrieval()`도 한 곳에서 실행합니다.

**리뷰어 · 후속 질문 💭**

그러면 exact/alias/pattern/FTS 모두 log와 threshold 형식이 하나의 code path를 지나겠네요.

**작업자 · 답변**

네. route 구현체는 log를 직접 쓰지 않습니다. selection 뒤의 accepted/rejected trace를 `UpdateDb`가
공통으로 기록하므로 audit schema의 route와 reason format도 유지됩니다. Update DB 8건과 전체 110건으로
기존 retrieval과 ingestion 회귀를 확인했습니다.

**리뷰어 · 확인 📌**

SQL 분리는 했지만 FTS fallback과 audit의 단일 기록 경로는 유지됐습니다.

## 승인

**리뷰어 · Approve ✅**

네 조회 route의 후보 수집을 Strategy로 분리하면서도 final selection, answer-type safety, audit log를
공통으로 유지했습니다. retrieval 순서·empty disable·injection contract가 test로 고정돼 승인합니다.
