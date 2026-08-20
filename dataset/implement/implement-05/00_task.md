# Implement-05 — preference export 전략 분리

## 개발 요청

SimPO·DPO·KTO adapter가 preference request를 학습 행으로 바꾸고 JSONL을 쓰는 흐름을
각자 갖고 있습니다. paired preference와 binary preference의 행 규칙을 분리하고,
공통 export 절차를 정리해 주세요.

## 완료 조건

- paired 전략은 chosen/rejected가 모두 있는 request만 한 행으로 만듭니다.
- binary 전략은 chosen·rejected가 있는 각각의 값을 독립 행으로 만듭니다.
- SimPO/DPO는 동일한 paired dataset·SimPO asset 생성 절차를 공유합니다.
- KTO는 paired training script를 만들지 않습니다.
- 기존 SimPO·DPO·KTO JSON/JSONL artifact와 manifest 결과가 같아야 합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/algorithms/preference.py`, `tests/test_pipeline*`에 한정합니다.
- training script 내용, config 파일 이름, artifact key, command 계약은 바꾸지 않습니다.

## 작업 단위

이번 작업은 행 변환을 `PreferenceRecordStrategy`로 분리하고, 공통 JSONL export를
Template Method로 모으는 behavior-preserving refactor입니다.
