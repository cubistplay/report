# Implement-02 — benchmark request 생성 책임 분리

## 개발 요청

`benchmarks.py`의 CounterFact, KnowEdit, MQuAKE, RippleEdits adapter를 정리해
주세요. 네 loader가 `CorrectionRequest` 생성, 공통 metadata 조립, answer type 주입을
각각 반복하고 있습니다.

## 완료 조건

- adapter별 입력 해석은 각 loader에 남깁니다.
- 공통 `CorrectionRequest` 생성과 metadata 규칙은 한 Factory에 둡니다.
- `subject`, `relation`, 추가 metadata, answer type의 우선순위가 한 곳에서 보입니다.
- Factory는 caller가 준 metadata를 수정하지 않습니다.
- semantic matcher가 없을 때와 있을 때의 answer type 처리 계약을 유지합니다.
- 네 adapter의 변환 결과는 이전과 같아야 합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` 두 commit으로 남깁니다.
- 변경은 `brainwash/benchmarks.py`와 `tests/test_benchmark_adapters*`에 한정합니다.
- input schema, CLI, output manifest 형식은 변경하지 않습니다.

## 작업 단위

네 loader가 공유하는 생성 규칙을 Factory로 모으는 behavior-preserving refactor입니다.
입력 해석과 공통 생성 책임을 구분하므로, 새 benchmark adapter를 추가할 때 metadata
규칙이 달라지는 위험을 줄일 수 있습니다.
