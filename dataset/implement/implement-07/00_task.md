# Implement-07 — memory retrieval Strategy 분리

## 개발 요청

`UpdateDb.retrieve_memory_edit()`에 exact, alias, pattern, FTS 후보 조회 SQL과 score 계산,
semantic rerank, answer-type 검증, audit log가 한 메서드에 섞여 있습니다. 조회 경로를
확장하거나 개별 SQL을 바꿀 때 selection 정책까지 함께 읽어야 하므로, 조회 후보 수집을
Strategy로 분리해 주세요.

## 완료 조건

- exact → alias → pattern → FTS의 기존 조회 우선순위를 유지합니다.
- 각 Strategy는 SQL prefilter와 초기 score만 담당합니다.
- semantic rerank, answer-type gate, threshold 결정, retrieval log는 `UpdateDb`에 둡니다.
- 빈 Strategy 목록은 의도적으로 memory lookup을 비활성화합니다.
- 주입한 Strategy가 원본 query와 normalized query를 모두 받는지 검증합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/memory/update_db.py`, `tests/test_update_db.py`에 한정합니다.
- 기존 retrieval route, trace 필드, FTS 오류 처리, Update DB schema는 바꾸지 않습니다.

## 작업 단위

이번 작업은 Memory Store 후보 조회라는 하나의 제어 흐름을 Strategy로 추출하는
behavior-preserving refactor입니다. 300줄을 넘지만 네 SQL 경로, shared candidate model,
constructor injection, rerank delegation, regression test를 하나의 reviewable 단위로 유지했습니다.
