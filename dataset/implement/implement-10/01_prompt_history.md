# 프롬프트 히스토리 — I-A10

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — in-memory lifecycle 정렬

**Developer**

> DB active_memory view는 valid_from/valid_until을 보는데 ledger는 status만 봐. future나
> expired correction이 active_memory.jsonl, conflict, promotion에 섞이면 안 돼. 시간 기준을
> test에서 고정할 수 있게 lifecycle 정책으로 정리하고 audit도 남겨줘.

**Codex**

> `MemoryLifecyclePolicy`가 record status와 validity window를 `MemoryLifecycleDecision`으로
> 평가하게 하겠습니다. ledger의 active snapshot, conflicts, exact index, promotion report는 이
> Policy 결과를 사용하고, lifecycle JSONL artifact에 reason을 남기겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 2026-08-21 기준으로 active 하나, future 하나, expired 하나, retired 하나를 같이 넣어봐.
> invalid timestamp도 조용히 active로 넘기지 말고 reason을 남기고, artifact에 lifecycle 결과와
> active snapshot이 같이 맞는지 확인해줘.

**작업 단계 — Red 테스트**

- injected `as_of`에서 active/future/expired/retired decision
- invalid `valid_from`의 explicit rejection reason
- lifecycle JSONL artifact와 empty active snapshot

```bash
python3 -m unittest tests.test_memory_ledger -q
# ImportError: MemoryLifecyclePolicy가 아직 없습니다.
```

Commit: `f930e71 test(implement-10): specify memory lifecycle policy`

## Prompt 3 — 기존 operational 흐름 검증

**Developer**

> 이건 active record 판단만 통일하는 거니까 record JSONL 형식이나 Update DB ingest는 바뀌면
> 안 돼. ledger와 Update DB, 전체 테스트까지 확인해줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 10 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 120 tests — OK
```

`MemoryLifecyclePolicy(as_of=...)`를 주입해 timestamp 비교를 deterministic하게 만들고,
`memory_lifecycle.jsonl`에 active 여부와 reason을 기록했습니다. full suite는 기존 sqlite connection
`ResourceWarning` 2건을 출력했으나 실패는 없었습니다.

Commit: `6fcf422 refactor(implement-10): centralize memory lifecycle policy`
