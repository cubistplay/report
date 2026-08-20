# R-A4 PR 대화 — SFT dataset summary artifact 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: `review/review-04-sft-summary` (`026760f`) · 최종 head: `1b5bba2`

## 스레드 1 — data loss를 숨기는 total count

**위치** `brainwash/algorithms/finetune.py` (`QLoRASFTAdapter.prepare`, 초기 PR 178-182행)  
**심각도** `blocking` · **Change Request**

**리뷰어 · 댓글 :thinking:**

summary의 `total_requests`에 `len(rows)`를 넘깁니다. `rows`는 target 없는 request를 skip한 뒤의
training output이라 total input이 아닙니다. 입력 2개 중 1개가 skip돼도 summary는 `1/1`과 ratio
`1.0`을 기록해서 data loss를 숨깁니다.

count가 의미하는 대상을 코드로 보면 차이가 명확합니다.

```python
# input 2건 중 target 없는 1건은 rows에 들어가지 않습니다.
SftDatasetSummary(total_requests=len(rows), training_rows=len(rows), ...)
# total_requests=1, usable_ratio=1.0  ← skip된 input이 사라짐

SftDatasetSummary(total_requests=len(requests), training_rows=len(rows), ...)
# total_requests=2, usable_ratio=0.5  ← 실제 input 대비 usable data
```

`len(requests)`를 total로 전달하고, target 하나가 없는 mixed input을 regression test로 고정해
주세요.

**작업자 · 답변 :speech_balloon:**

맞습니다. summary 생성 시점에 training output count를 input count로 잘못 사용했습니다.
`request_count=len(requests)`로 바꾸고, total·row·skip·ratio를 같이 확인하는 test를 추가하겠습니다.

**리뷰어 · 후속 :mag:**

이렇게 하면 `training_rows / total_requests`가 실제 usable ratio가 됩니다. skip count만 따로
보는 것보다 summary 한 개로 data quality를 판단할 수 있겠습니다.

**작업자 · 반영**

`1b5bba2 fix(review-04): publish SFT dataset summary`

summary total은 전체 `requests` 길이를 사용하도록 수정했습니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. skipped request가 summary 분모에 남아 data loss가 보입니다.

## 스레드 2 — 생성했지만 manifest에서 찾을 수 없는 artifact

**위치** `brainwash/algorithms/finetune.py` (`AlgorithmRunResult` 생성, 초기 PR 205-211행)  
**심각도** `important` · **Change Request**

**리뷰어 · 댓글 :warning:**

`sft_dataset_summary.json`은 생성하지만 `artifacts` dict에는 없습니다. pipeline consumer는
`manifest.json`의 artifacts를 기준으로 run output을 탐색하므로, 파일이 있어도 stable contract가
아닙니다.

`AlgorithmRunResult`가 manifest로 공개하는 것은 `artifacts` mapping뿐입니다. 따라서 아래처럼 파일을
쓰기만 하면 directory를 직접 스캔하는 consumer만 찾을 수 있고, stable key가 없습니다.

```python
summary_path.write_text(summary_json)
AlgorithmRunResult(artifacts={})
# manifest에 summary 경로 없음

AlgorithmRunResult(artifacts={"dataset_summary": str(summary_path)})
# manifest consumer가 dataset_summary key로 찾을 수 있음
```

[JSON Schema object 문서](https://json-schema.org/understanding-json-schema/reference/object)는 object의
property가 기본적으로는 없어도 유효하며, 외부 계약에서 존재를 보장하려면 required field로 다뤄야 함을
설명합니다. 이 manifest에 JSON Schema를 도입하자는 뜻은 아닙니다. 다만 `dataset_summary`처럼 consumer가
반드시 찾아야 하는 값은 file 생성만 믿지 말고 manifest key 존재를 test로 고정해야 한다는 근거입니다.

`dataset_summary` key로 summary path를 반환해 주세요. summary file 존재와 manifest value를 함께
검증하는 test도 필요합니다.

**작업자 · 답변 :memo:**

동의합니다. output directory에 파일을 쓰는 것만으로는 pipeline artifact가 되지 않습니다.
`dataset_summary`를 result artifact에 추가하고 test에서 manifest 경로까지 확인하겠습니다.

**리뷰어 · 후속 :eyes:**

config에 summary를 복사하는 것과 artifact path를 공개하는 것은 다른 책임입니다. config는 trainer
설정에 가깝고, artifact는 외부 tool이 file을 찾는 계약이므로 둘 다 필요합니다.

**작업자 · 반영 :+1:**

`1b5bba2`에 `dataset_summary: str(summary_path)`를 추가했습니다.

**리뷰어 · 확인 :white_check_mark:**

manifest에서 summary artifact를 찾을 수 있고, file 생성과 discovery가 모두 보호됩니다.

## 스레드 3 — summary regression test 부재

**위치** `brainwash/algorithms/finetune.py` (`SftDatasetSummary`)  
**심각도** `important`

**리뷰어 · 댓글 :test_tube:**

새 summary class는 count, ratio, JSON shape, run note를 동시에 결정하지만 해당 계약 test가 없습니다.
target 하나가 없는 input을 실제 pipeline으로 준비해 summary JSON과 manifest를 읽는 test를 추가해
주세요.

**작업자 · 답변 :speech_balloon:**

`tests/test_pipeline.py`에 QLoRA override를 사용한 test를 추가하겠습니다. `2 / 1 / 1 / 0.5`,
manifest artifact 경로, `Prepared 1/2 usable SFT rows.` note까지 확인하겠습니다.

**리뷰어 · 후속 :bulb:**

direct helper test보다 pipeline test가 적절합니다. summary path가 `AlgorithmRunResult`와 manifest를
거쳐도 유지되는지 확인할 수 있기 때문입니다.

**작업자 · 반영**

`594baf5 test(review-04): specify SFT dataset summary artifacts`

test는 initial implementation에서 Red였고 `1b5bba2` 후 pipeline 6건과 전체 99건이 통과했습니다.

**리뷰어 · 확인 :white_check_mark:**

count semantics와 artifact discoverability가 pipeline 수준 regression test로 고정됐습니다.

## 스레드 4 — empty dataset의 ratio 정책

**위치** `SftDatasetSummary.usable_ratio`  
**심각도** `question`

**리뷰어 · 질문 :question:**

모든 request가 skip된 경우 ratio를 `0.0`으로 둡니다. `None`으로 “계산 불가”를 표현할 수도 있는데,
0.0을 선택한 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

이 summary에서 ratio는 “입력 중 usable row 비율”이라 input이 없거나 usable row가 없으면 0.0으로
두었습니다. `is_empty`와 note가 empty 상태를 별도로 설명하므로 numeric consumer는 0.0을 그대로
집계할 수 있습니다.

**리뷰어 · 확인 :+1:**

numeric schema를 유지하면서 empty 상태를 별도 field/note로 설명하니 consumer 처리도 단순합니다.
변경 요청은 없습니다.

## 스레드 5 — QLoRA와 LoRA의 summary 공유

**위치** `QLoRASFTAdapter`, `LoRASFTAdapter`  
**심각도** `question`

**리뷰어 · 질문 :eyes:**

LoRA adapter는 QLoRA adapter를 상속하면서 4bit 옵션만 바꿉니다. summary도 두 adapter에서 같은
형식을 쓰는 것이 의도인가요?

**작업자 · 답변 :speech_balloon:**

네. input filtering과 train JSONL 생성은 같은 SFT preparation 절차이고, hardware quantization만
다릅니다. summary는 data preparation 결과라 4bit 여부와 독립적으로 공유하는 것이 맞습니다.

**리뷰어 · 확인 :white_check_mark:**

공유 base adapter에 summary를 둔 책임 경계가 자연스럽습니다. adapter별로 중복 summary를 만들
필요는 없습니다.

## 최종 승인

**리뷰어 · Approve :tada:**

SFT summary가 실제 input 대비 usable data를 보고하고, manifest artifact를 통해 발견 가능하게
노출되는지 확인했습니다. mixed input pipeline test가 count·ratio·artifact·note를 고정합니다.

focused pipeline test 6건과 전체 test 99건 통과를 확인하여 승인합니다.
