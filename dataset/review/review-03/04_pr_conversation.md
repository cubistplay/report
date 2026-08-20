# R-A3 PR 대화 — benchmark provenance metadata 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: `review/review-03-benchmark-provenance` (`7d612cd`) · 최종 head: `5a6f568`

## 스레드 1 — provenance ownership이 extra metadata에 밀림

**위치** `brainwash/benchmarks.py` (`BenchmarkRequestFactory.create`, 초기 PR 121-126행)  
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

`provenance_metadata`를 먼저 merge한 뒤 `extra_metadata`를 합칩니다. adapter 또는 향후 caller가
`benchmark_source`를 넣으면 원본 provenance가 바뀝니다. provenance는 request의 출처를 설명하는
audit field라 임의 metadata보다 권위가 높아야 합니다.

Python의 dictionary display에서 `**` unpacking은 뒤에 오는 값이 같은 key의 앞 값을 대체합니다.
따라서 현재 순서는 아래처럼 실제 출처를 잃습니다.

```python
{**{"benchmark_source": "counterfact"}, **{"benchmark_source": "not-the-source"}}
# {"benchmark_source": "not-the-source"}
```

반대로 provenance를 마지막에 두면 factory가 보장해야 하는 출처 좌표가 남습니다. 이 동작은
[Python dictionary display 문서](https://docs.python.org/3/reference/expressions.html#dictionary-displays)의
unpacking 규칙과도 일치합니다.

`extra_metadata`를 먼저 합친 뒤 provenance를 마지막에 적용해 주세요. `benchmark_source` overwrite를
시도해도 실제 source가 남는 regression test도 필요합니다.

**작업자 · 답변 :speech_balloon:**

맞습니다. factory가 output metadata를 만들 때 provenance ownership을 명확히 하지 않았습니다.
adapter-specific metadata는 보조 정보로 두고 source coordinate는 마지막에 적용하겠습니다.

**리뷰어 · 후속 :eyes:**

`subject`와 `relation`도 factory가 보장하는 metadata라 지금처럼 caller 값을 덮어쓰는 경계와
일관됩니다. provenance도 같은 방식으로 보면 되겠습니다.

**작업자 · 반영**

`5a6f568 fix(review-03): preserve benchmark provenance`

metadata merge 순서를 `extra → subject/relation → provenance`로 변경했습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. provenance가 source of truth로 남고, test가 같은 key overwrite를 막습니다.

## 스레드 2 — MQuAKE의 원본 rewrite 위치가 사라짐

**위치** `brainwash/benchmarks.py` (`load_mquake_requests`, 초기 PR 218행)  
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :warning:**

`benchmark_rewrite_index`에 `len(requests)`를 넣으면 전체 request 누적 순서가 기록됩니다. 이 값은
원본 `requested_rewrite` list의 위치가 아닙니다. 앞 rewrite가 target 누락으로 skip되면 두 번째
source rewrite가 `#0`처럼 보입니다.

이미 `enumerate()`의 `index`가 있으니 그 값을 provenance에 넘겨 주세요. source coordinate는
processing 결과가 아니라 원본 dataset을 다시 찾을 수 있어야 합니다.

**작업자 · 답변 :memo:**

동의합니다. `len(requests)`는 adapter output 순서일 뿐 source coordinate가 아닙니다. loop의
`index`를 사용하고, 첫 rewrite가 skip된 뒤 두 번째가 `#1`로 남는 test를 추가하겠습니다.

**리뷰어 · 후속 :mag:**

case ID도 raw int와 string이 섞이면 reference consumer가 타입별 처리를 해야 합니다. row boundary에서
string으로 정규화하면 `mquake:17#1` format도 안정적입니다.

**작업자 · 반영 :+1:**

같은 `5a6f568`에서 `case_id`를 string으로 정규화했고, MQuAKE provenance에는 `index`를 전달하도록
바꿨습니다.

**리뷰어 · 확인 :white_check_mark:**

좋습니다. source row를 다시 찾는 좌표와 adapter 출력 순서가 분리됐습니다.

## 스레드 3 — provenance contract test 부재

**위치** `brainwash/benchmarks.py` (`BenchmarkProvenance`, `BenchmarkRequestFactory`)  
**심각도** `important`

**리뷰어 · 댓글 :test_tube:**

신규 metadata는 downstream report와 audit log가 읽는 계약인데 test가 없습니다. 첫 스레드의
overwrite, 두 번째 스레드의 skipped rewrite, numeric case ID/reference 세 경우를 함께 고정해
주세요.

**작업자 · 답변 :speech_balloon:**

`tests/test_benchmark_adapters.py`에 factory-level overwrite test와 MQuAKE loader regression test를
추가하겠습니다. loader test가 case ID, rewrite index, `benchmark_reference`까지 확인하게 하겠습니다.

**리뷰어 · 후속 :bulb:**

reference만 확인하면 index와 case ID 중 하나가 우연히 맞을 수 있습니다. 세 metadata field도
각각 assertion으로 남겨 주세요.

**작업자 · 반영**

`d39e254 test(review-03): specify benchmark provenance contracts`

두 test는 initial implementation에서 Red였고, source field·index·reference를 각각 확인합니다.
`5a6f568` 적용 후 focused 11건과 전체 98건이 통과했습니다.

**리뷰어 · 확인 :white_check_mark:**

metadata와 사람이 읽는 reference를 각각 검증해서 provenance contract가 분명해졌습니다.

## 스레드 4 — single-rewrite dataset의 null index

**위치** `BenchmarkProvenance.to_metadata`  
**심각도** `question`

**리뷰어 · 질문 :question:**

CounterFact와 KnowEdit은 `benchmark_rewrite_index=None`을 metadata에 남깁니다. key를 아예 빼는
방식도 가능한데, null을 유지한 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

같은 provenance schema를 유지하고, null은 “rewrite list 안의 위치가 적용되지 않는다”는 명시적
표현으로 두었습니다. consumer가 key 존재 여부가 아니라 value만 보면 됩니다.

**리뷰어 · 확인 :+1:**

동의합니다. dataset별로 metadata shape가 달라지는 것보다 schema가 일정한 편이 report 처리에
유리합니다. 변경 요청은 없습니다.

## 스레드 5 — request ID와 provenance reference의 역할 분리

**위치** `BenchmarkProvenance.reference`  
**심각도** `question`

**리뷰어 · 질문 :eyes:**

기존 request ID에도 dataset 이름이 들어가는데 `benchmark_reference`를 따로 두는 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

request ID는 pipeline이 만든 실행 단위이고, provenance reference는 원본 row 좌표입니다. 특히
MQuAKE는 한 case에서 여러 rewrite가 나와 request ID 순번만으로 skipped rewrite의 원래 위치를
알 수 없습니다.

**리뷰어 · 확인 :white_check_mark:**

역할이 다르다는 설명이 명확합니다. report에서 origin을 표시할 때 reference를 쓰는 경계가
자연스럽습니다.

## 최종 승인

**리뷰어 · Approve :tada:**

provenance ownership, MQuAKE source rewrite coordinate, stable case ID와 regression coverage를
확인했습니다. source coordinate와 adapter output 순서를 분리했고, metadata overwrite도 막혔습니다.

focused benchmark adapter test 11건과 전체 test 98건 통과를 확인하여 승인합니다.
