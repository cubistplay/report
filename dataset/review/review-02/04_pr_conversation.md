# R-A2 PR 대화 — conversation resolution history 검토

> Synthetic GitHub artifact: true  
> 최초 검토 head: `235cb0c` · 최종 head: `ef03ac9`

## 스레드 1 — internal history list 노출

**위치** `brainwash/conversation.py` (`ConversationResolver.history`, 초기 PR 69-72행)  
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

`history`가 `_history` list 자체를 반환합니다. caller가 다음처럼 실행하면 resolver가 기록한
audit trail을 임의로 지우거나 가짜 record를 넣을 수 있습니다.

```python
resolver.history.clear()
resolver.history.append(fake_record)
```

조회 API는 resolver의 내부 collection을 소유하지 않아야 합니다. 읽기 전용 tuple snapshot을
반환하고 그 동작을 test로 고정해 주세요.

**작업자 · 답변 :speech_balloon:**

동의합니다. history는 관찰용 결과인데 list 참조를 넘겨 내부 state를 수정할 수 있게 했습니다.
public API는 `tuple(self._history)`를 반환하고, clear는 명시적인 `clear_history()`만 사용하도록
바꾸겠습니다.

**리뷰어 · 후속 :eyes:**

이렇게 하면 caller가 받은 과거 history도 이후 resolver가 새 query를 처리해도 길이나 내용이
바뀌지 않는 snapshot이 됩니다. log 조회 API에 더 맞는 의미입니다.

**작업자 · 반영**

`ef03ac9 fix(review-02): protect resolution history snapshots`

`history` 반환형을 `tuple[ConversationResolutionRecord, ...]`로 바꿨습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. append/clear가 불가능하고 resolver 내부 기록은 명시적 API로만 변경됩니다.

## 스레드 2 — frozen record 안의 mutable subject list

**위치** `brainwash/conversation.py` (`ConversationResolutionRecord.known_subjects`, 초기 PR 31행)  
**심각도** `important`

**리뷰어 · 댓글 :warning:**

record가 `@dataclass(frozen=True)`이지만 `known_subjects`는 `list[str]`입니다. Python dataclass의
`frozen=True`은 field 재할당을 막는 것이지, field가 가리키는 list의 `.append()`까지 막지는
않습니다. 즉 아래 코드는 초기 PR에서 성공합니다.

```python
record.known_subjects.append("Injected subject")
```

[Python dataclasses 문서](https://docs.python.org/3/library/dataclasses.html#frozen-instances)도
`frozen=True`을 field assignment가 예외가 되는 read-only 동작으로 설명합니다. audit snapshot에는
mutable list 대신 tuple이 적절해 보입니다.

**작업자 · 답변 :memo:**

맞습니다. record 자체가 frozen이라는 표기와 실제 nested mutation 가능성이 어긋납니다. 생성
시점에 `tuple(subjects)`로 저장하고 type도 `tuple[str, ...]`로 바꾸겠습니다.

**리뷰어 · 후속 :bulb:**

`to_dict()`는 JSON artifact용 경계이므로 tuple을 그곳에서만 list로 바꾸면 in-memory contract와
export format을 함께 지킬 수 있겠습니다.

**작업자 · 반영 :+1:**

그대로 반영했습니다. record는 immutable tuple을 보관하고 `to_dict()["known_subjects"]`만
JSON-compatible list를 반환합니다.

**리뷰어 · 확인 :white_check_mark:**

좋습니다. record의 표현 불변성과 JSON 직렬화 책임이 분리됐습니다.

## 스레드 3 — history 계약의 회귀 테스트

**위치** `brainwash/conversation.py` (`ConversationResolutionRecord`, `history`, `clear_history`)  
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :test_tube:**

신규 history feature에 대한 test가 없습니다. 최소한 다음 세 계약은 실제 동작으로 고정해
주세요.

1. caller가 `history`에 append하지 못한다.
2. 첫 resolution의 subject snapshot은 이후 subject 추가로 바뀌지 않는다.
3. history를 지운 뒤에도 resolver는 기존 subject로 follow-up을 해석한다.

**작업자 · 답변 :speech_balloon:**

세 경우를 `tests/test_benchmark_adapters.py`에 추가하겠습니다. 기존 conversation resolver
integration test가 같은 파일에 있으므로, 새 test도 그 경로에 두는 것이 자연스럽습니다.

**리뷰어 · 후속 :mag:**

두 번째 test는 tuple type만 보는 대신, 첫 record를 만든 뒤 두 번째 subject를 등록하는 순서를
포함해 주세요. 그래야 실제 turn snapshot임을 검증합니다.

**작업자 · 반영**

`1acd9de test(review-02): specify resolution history snapshots`

세 test를 추가했습니다. 이 test commit은 initial implementation에서 history list와 nested list를
노출하므로 Red 상태였고, 다음 `ef03ac9`에서 Green으로 만들었습니다.

`python3 -m unittest tests.test_benchmark_adapters -q` 9건과 전체 96건을 통과했습니다.

**리뷰어 · 확인 :white_check_mark:**

세 API 경계가 모두 regression test로 남았습니다. 특히 subject 등록 순서가 포함되어 snapshot
의미를 확인할 수 있습니다.

## 스레드 4 — clear는 context reset이 아님

**위치** `brainwash/conversation.py` (`ConversationResolver.clear_history`)  
**심각도** `question`

**리뷰어 · 질문 :question:**

`clear_history()`가 history만 비우고 `subjects`는 유지합니다. 이름만 보면 새 대화를 시작하는
것처럼 오해할 수 있는데, 이 동작은 의도된 것인가요?

**작업자 · 답변 :speech_balloon:**

의도했습니다. history는 관찰·디버깅 로그이고 subjects는 현재 conversation의 rewrite context라
서로 수명이 다릅니다. 새 conversation context가 필요하면 별도 resolver instance를 만들도록
두었습니다.

**리뷰어 · 후속 :+1:**

그 구분이면 `clear_history`라는 이름도 정확합니다. reset semantics를 몰래 섞지 않은 점이
좋고, thread 3의 test가 이 계약을 보호하니 추가 변경은 필요 없습니다.

## 스레드 5 — 모든 결과 상태를 기록하는 범위

**위치** `brainwash/conversation.py` (`ConversationResolver.resolve`)  
**심각도** `question`

**리뷰어 · 질문 :eyes:**

성공한 rewrite뿐 아니라 `unchanged`, `unresolved`, `ambiguous`도 `_record()`를 거칩니다.
실패 상태까지 남기는 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

이력의 목적은 “어떤 rewrite가 일어났나”뿐 아니라 “왜 rewrite하지 않았나”도 확인하는 것입니다.
특히 ambiguous 결과는 후보 subject가 둘 이상이었던 시점의 근거가 있어야 follow-up 문제를
재현할 수 있습니다.

**리뷰어 · 확인 :white_check_mark:**

동의합니다. 상태별 record를 별도 class로 나누기보다 같은 resolution schema에 reason을 두는
편이 query 처리 흐름을 한 번에 읽을 수 있습니다.

## 최종 승인

**리뷰어 · Approve :tada:**

초기 PR의 history container와 frozen record 내부 list가 실제로 수정 가능한 문제를 확인했고,
tuple snapshot과 regression test로 해결됐습니다. log/context 수명과 실패 상태 기록 범위도
명확히 논의했습니다.

focused test 9건과 전체 test 96건 통과를 확인하여 승인합니다.
