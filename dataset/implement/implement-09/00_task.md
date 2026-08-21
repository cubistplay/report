# Implement-09 — memory match Strategy 분리

## 개발 요청

`MemoryEditStore.evaluate_trigger()`는 exact/paraphrase 확인, subject-aware token overlap,
best candidate 선택, threshold, answer-type gate를 하나의 loop에서 처리합니다. 후보 근거를 만드는
match 규칙과 최종 memory safety gate를 분리해 주세요.

## 완료 조건

- exact/paraphrase match가 subject-aware token overlap보다 먼저 적용됩니다.
- match Strategy는 candidate prompt, score, match 근거만 반환합니다.
- threshold와 answer-type validation은 `MemoryEditStore`에 유지합니다.
- 빈 Strategy 목록은 의도적으로 memory triggering을 비활성화합니다.
- custom Strategy가 raw query와 각 `MemoryEdit`을 받는지 검증합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/algorithms/memory_edit.py`, `tests/test_memory_edit_runtime.py`에 한정합니다.
- existing verifier mode, trigger reason, strict fallback, augmented prompt 결과를 바꾸지 않습니다.

## 작업 단위

이번 작업은 in-memory Memory Store의 후보 탐색을 Strategy와 `MemoryMatchPolicy`로 분리하는
behavior-preserving refactor입니다. 335줄 안에서 candidate evidence, two match routes, common
safety gate, extension contract test를 하나의 reviewable 단위로 완료합니다.
