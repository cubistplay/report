# Implement-10 — memory lifecycle Policy 통합

## 개발 요청

Memory DB의 `active_memory` view는 status, `valid_from`, `valid_until`을 함께 적용하지만,
in-memory `MemoryLedger.active_records()`는 status만 확인합니다. future/expired record가 active
snapshot과 artifact에 포함되지 않도록 lifecycle 판단을 한 정책으로 통합해 주세요.

## 완료 조건

- status, start time, end time을 평가해 active 여부와 이유를 반환합니다.
- future, expired, retired, invalid timestamp record를 active snapshot에서 제외합니다.
- `as_of`를 주입해 시간 경계 test를 deterministic하게 작성합니다.
- lifecycle audit artifact에 record별 active 상태와 reason을 남깁니다.
- 기존 conflicts, exact index, promotion report가 active lifecycle 결과를 사용합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/memory/ledger.py`, `tests/test_memory_ledger.py`에 한정합니다.
- Memory Record JSONL schema와 existing `write_artifacts()` key는 유지하고 lifecycle key만 추가합니다.

## 작업 단위

이번 작업은 DB와 in-memory ledger 사이의 active-record lifecycle 계약을 `MemoryLifecyclePolicy`로
공유하는 구조 개선입니다. 231줄 안에서 lifecycle Decision Value Object, deterministic clock,
active filtering, audit artifact, temporal regression test를 하나의 reviewable 단위로 완료합니다.
