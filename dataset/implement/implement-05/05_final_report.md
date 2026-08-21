# I-A5 개발 활동 보고서 — preference export 전략 분리

## 1. 배경

SimPO와 DPO는 chosen/rejected pair로 학습 데이터를 만들고, KTO는 각 응답을 binary
label 행으로 만듭니다. 그러나 adapter마다 request 순회와 JSONL 기록 절차가 분산되어
있어 새로운 preference objective를 추가하거나 artifact 규칙을 바꿀 때 행 의미와
출력 절차를 함께 추적해야 했습니다.

이번 변경은 학습 행의 의미를 record Strategy로 분리하고, dataset JSONL export를 base
adapter로 모았습니다. SimPO/DPO가 공유하던 paired asset 생성도 별도 Template Method로
정리했으며 기존 파일·command 계약은 유지했습니다.

## 2. Commit 및 PR 경계

- base: `main` / `067cbce8b321fb13593ce38de54ea3125e301a83`
- Red 테스트: `414878049cd049d26e04277c824da0998fafa9dd`
  `test(implement-05): specify preference record strategies`
- 최초 PR 및 최종 head: `cd8199fed9ea342109640a3e71c51de6c8a842b6`
  `refactor(implement-05): separate preference export strategies`
- 최종 `main`: `cd8199fed9ea342109640a3e71c51de6c8a842b6`

최초 head에서 record 전략의 의미, shared paired asset 범위, 공통 export의 artifact
보존을 검토했습니다. 코드 결함은 발견되지 않아 Change Request나 후속 commit은 만들지
않았습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 아직 없는 `PairedPreferenceRecords`와 `BinaryPreferenceRecords` import에서
실패했습니다. paired skip count, binary 독립 행, DPO의 paired asset 재사용, KTO의
paired asset 미생성을 먼저 명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_pipeline_preference_export -q
# Ran 4 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 90 tests — OK
```

또한 동일한 preference request를 SimPO·DPO·KTO adapter에 전달해 manifest와 JSON/JSONL
artifact를 만들고, 임시 output 경로만 정규화해 변경 전 commit과 비교했습니다. `diff`
결과가 없어 세 adapter의 출력 계약이 보존됐음을 확인했습니다.

## 4. 구조 개선

`PreferenceRecordStrategy`는 request가 학습 행으로 변환되는 정책을 표현합니다.
`PairedPreferenceRecords`는 완전한 pair만 생성하고 제외된 request 수를 기록하며,
`BinaryPreferenceRecords`는 available chosen/rejected를 독립 label 행으로 만듭니다.

`PreferenceExportAdapter`는 strategy 실행과 JSONL 기록을 공통으로 처리합니다.
`PairedPreferenceExportAdapter`는 paired dataset, SimPO training script,
`simpo_config.json` 생성을 공유합니다. 따라서 DPO는 shared asset 위에 자신의 config만
추가하고, KTO는 binary dataset과 KTO config만 생성합니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 272줄 |
| 삭제 | 47줄 |
| 합계 | 319줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/algorithms/preference.py`와
`tests/test_pipeline_preference_export.py`입니다. 319줄 안에서 Strategy·Template Method
도입, 중복 export 제거, adapter별 회귀 테스트, artifact 동등성 검증을 하나의 검토 단위로
완결했습니다.

## 6. 리뷰 결과

리뷰에서는 paired/binary 행의 의미, DPO가 공유하는 asset과 실행 책임의 경계, KTO가
공통 export를 쓰면서 paired asset을 만들지 않는지를 확인했습니다. strategy 테스트,
adapter 통합 테스트, 변경 전후 artifact 비교로 검증했고 추가 코드 변경 없이 승인되었습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
