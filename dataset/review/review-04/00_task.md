# Review-04 — SFT dataset summary artifact 검토

## 검토 요청

QLoRA/LoRA SFT adapter가 training data를 준비할 때 전체 입력, usable row, skip 수를
summary JSON으로 남기는 PR을 검토해 주세요. 초기 PR은 기존 suite를 통과하지만 summary count와
manifest artifact 계약을 직접 검증하는 test는 없습니다.

## 검토 범위

- `brainwash/algorithms/finetune.py`의 `SftDatasetSummary`
- skip된 request를 포함한 dataset count와 usable ratio
- summary file 생성과 `AlgorithmRunResult.artifacts` 노출
- SFT pipeline regression test

## 완료 조건

- total request와 emitted training row를 혼동하지 않는지 확인합니다.
- 생성한 summary가 pipeline manifest에서 발견 가능한지 확인합니다.
- Change Request는 regression test와 후속 code commit으로 해결합니다.
- focused pipeline test와 전체 suite를 통과시킵니다.

## 제한

- 변경은 `brainwash/algorithms/finetune.py`, `tests/test_pipeline.py`에 한정합니다.
- 최초 PR은 기존 full suite를 통과하는 완결된 기능이어야 합니다.
- 리뷰 시작 뒤에는 reviewed commit을 수정하지 않고 선형 response commit만 추가합니다.
