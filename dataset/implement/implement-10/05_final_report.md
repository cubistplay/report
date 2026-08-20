# I-A10 개발 활동 보고서 — memory lifecycle Policy 통합

## 1. 배경

Memory DB의 `active_memory` view는 active status와 validity window를 함께 적용하지만, in-memory
Memory Ledger는 status만 보고 active record를 선택했습니다. future/expired record가 active snapshot,
conflict, index, promotion report에 포함될 수 있어 DB와 operational artifact의 의미가 달랐습니다.

이번 변경에서는 status와 time window를 `MemoryLifecyclePolicy`로 통합하고, 결과를
`MemoryLifecycleDecision`으로 기록했습니다. Ledger는 Policy를 통해 active record를 사용하고
`memory_lifecycle.jsonl` audit artifact를 추가합니다.

## 2. Commit 및 PR 경계

- base: `main` / `1fe0233c8be14d36b847804b5e820d98b9ef2ebf`
- Red 테스트: `700d61c4516730c30c57d1f0b5a402f2d66b9917`
  `test(implement-10): specify memory lifecycle policy`
- 최초 PR 및 최종 head: `8291841b976624af414c2dfd96d7e2b596115ea8`
  `refactor(implement-10): centralize memory lifecycle policy`
- 최종 `main`: `8291841b976624af414c2dfd96d7e2b596115ea8`

최초 head에서 DB/in-memory window semantics, invalid timestamp의 안전한 처리, lifecycle audit과
existing ingestion boundary를 검토했습니다. 코드 결함은 발견되지 않아 Change Request나 후속 commit은
만들지 않았습니다.

## 3. TDD 및 검증

Red 테스트는 존재하지 않는 `MemoryLifecyclePolicy` import에서 실패했습니다. injected `as_of`에서
active/future/expired/retired decision, invalid timestamp rejection, lifecycle artifact와 active snapshot
일치를 먼저 명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 10 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 120 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 구조 개선

`MemoryLifecyclePolicy`는 status, valid_from, valid_until을 one policy로 평가합니다. start boundary는
inclusive, end boundary는 exclusive로 DB view와 동일하게 적용하고, invalid timestamp는 explicit inactive
reason으로 남깁니다.

`MemoryLifecycleDecision`은 record ID, active 여부, reason, as-of timestamp를 Value Object로
직렬화합니다. `MemoryLedger`는 Policy의 decision으로 active record를 고르고, active snapshot,
conflicts, exact index, promotion report와 lifecycle JSONL artifact를 만듭니다.

optional `as_of`와 constructor-level Policy injection은 test·backfill·historical audit에서 deterministic
결과를 제공하지만 default runtime은 UTC now를 사용하므로 기존 호출 형식은 유지됩니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 206줄 |
| 삭제 | 3줄 |
| 합계 | 209줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/memory/ledger.py`, `tests/test_memory_ledger.py`입니다. 209줄 안에서 lifecycle
Policy, Decision Value Object, active filtering, audit artifact, deterministic temporal test를 하나의
coherent refactor로 완료했습니다.

## 6. 리뷰 결과

리뷰에서는 DB view와 in-memory active window semantics, inactive reason/audit 책임, injected time과
existing Update DB ingestion 보존을 확인했습니다. 세 material thread와 ledger·Update DB·전체 suite
검증을 근거로 승인했습니다.
