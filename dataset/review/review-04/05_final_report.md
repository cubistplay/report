# R-A4 최종 리뷰 보고서 — SFT dataset summary artifact

## 1. 검토 배경

대상 PR은 QLoRA/LoRA SFT preparation 결과를 summary JSON, config, note로 남기는 기능을
추가했습니다. 초기 head는 전체 test 98건을 통과했지만, summary 분모가 실제 input인지와 생성한
file이 manifest에서 발견되는지는 검증하지 않았습니다.

리뷰에서는 data quality summary가 training output만 예쁘게 보이는 보고서가 아니라, skip으로
사라진 request까지 보여 주는 운영 계약인지 검토했습니다.

## 2. Commit 및 PR 경계

- base: `43064678c1ccf8126493b12908b586fb5b780910`
- 최초 PR branch/head: `review/review-04-sft-summary` /
  `65f2110647de0e1d16f8759b2b4c03e0d7f53f4d`
- 리뷰 반영 테스트: `f45a88958fe7a9f97cfb00d51bfd04d8be1cc0a2`
  `test(review-04): specify SFT dataset summary artifacts`
- 리뷰 반영 수정 및 최종 head: `29ec5fe6068970e75797126079e849ff1ce49253`
  `fix(review-04): publish SFT dataset summary`

리뷰 뒤 최초 PR commit은 변경하지 않았습니다. regression test와 response code를 새 commit으로
누적해 `main`의 선형 이력을 유지했습니다.

## 3. 발견 사항과 반영 결과

| 심각도 | 발견 사항 | 영향 | 반영 |
| --- | --- | --- | --- |
| blocking | total에 emitted row count 사용 | skip된 input이 숨겨져 usable ratio 과대 표시 | `len(requests)` 사용 |
| important | summary file을 artifacts에 미등록 | manifest consumer가 file을 발견할 수 없음 | `dataset_summary` artifact 추가 |
| important | summary contract test 부재 | count·ratio·manifest 회귀 탐지 불가 | mixed input pipeline test 추가 |
| question | empty ratio 0.0 | no-input/empty-data 표현 정책 | numeric 0.0과 empty note 유지 |
| question | LoRA/QLoRA summary 공유 | adapter별 summary 책임 | shared preparation 결과로 확인 |

## 4. 리뷰 품질과 협업

첫 Change Request는 “2개 입력 중 1개 skip”이라는 concrete case로 `1/1` ratio가 잘못되는 것을
설명했습니다. `len(rows)`와 `len(requests)`의 차이가 단순 naming 문제가 아니라 data loss를
운영자가 놓치게 하는 오류라는 근거를 제시했습니다.

정보 제공으로 input count와 emitted row count를 같은 예시로 비교했습니다. input 2개에서 row가 1개이면
`total_requests=len(rows)`는 `1/1`을 기록하지만, `total_requests=len(requests)`는 `1/2`로 skip을
드러냅니다. 리뷰어가 count semantics와 수정 이유를 링크 없이 확인할 수 있는 코드 사례입니다.

두 번째는 output directory의 file 존재와 manifest artifact discoverability를 구분했습니다.
수정 방향으로 `dataset_summary` key를 명시했고, pipeline test가 summary file과 manifest path를
함께 확인하게 했습니다.

이때 file write와 artifact publication이 별개라는 정보도 공유했습니다. `summary_path.write_text(...)`
만으로는 manifest에 경로가 생기지 않으며, `artifacts["dataset_summary"] = str(summary_path)`가 있어야
pipeline consumer가 stable key로 결과를 찾을 수 있습니다.

[JSON Schema object 문서](https://json-schema.org/understanding-json-schema/reference/object)는 object의
property가 기본적으로 optional이고, 존재를 보장하려면 required contract가 필요하다고 설명합니다. 이 PR은
JSON Schema를 추가하지 않지만, 같은 원리로 summary artifact가 manifest에 실제로 존재하는지 pipeline test로
검증하도록 안내했습니다.

empty ratio와 LoRA/QLoRA 공유는 결함으로 과장하지 않고, data preparation schema와 adapter 책임
경계를 확인하는 질문으로 다뤘습니다.

## 5. 검증

```bash
python3 -m unittest discover -s tests -q
# Ran 98 tests — OK  (initial PR)

python3 -m unittest tests.test_pipeline -q
# Ran 6 tests — OK

python3 -m unittest discover -s tests -q
# Ran 99 tests — OK  (final)
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 6. 변경 범위

| 구간 | 파일 | 추가 | 삭제 | 합계 |
| --- | --- | ---: | ---: | ---: |
| 최초 PR | `brainwash/algorithms/finetune.py` | 53 | 2 | 55 |
| 리뷰 반영 | `brainwash/algorithms/finetune.py`, `tests/test_pipeline.py` | 29 | 2 | 31 |

초기 변경량 68줄은 Review PR의 일반적인 50~200줄 범위 안입니다. SFT data summary라는 하나의
기능을 검토했고, 반영 범위도 count·artifact·test 계약에 한정했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최종 변경 파일은 `black --check`를 통과했고, 재작성 전후 변경 Python 파일의 AST가 동일함을 확인했습니다. `#` 주석과 inline comment는 코드에서 제거했으며, 새 docstring은 추가하지 않았습니다.

