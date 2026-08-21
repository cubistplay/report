# 프롬프트 히스토리 — I-A6

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — promotion 기준 분리

**Developer**

> memory ledger에서 같은 subject/relation record가 일정 수 이상이면 training candidate로
> 올리는데, target이 서로 다르거나 high-risk면 그냥 올리면 안 될 것 같아. ledger가 다
> 판단하지 않게 기준을 따로 빼고, report에서 왜 막혔는지도 보이게 해줘.

**Codex**

> ledger는 active record와 artifact를 유지하고, `PromotionPolicy`가 cluster별 eligibility를
> 판단하도록 분리하겠습니다. 최소 건수, target 충돌, high-risk 검토를 각각 Rule로 두고
> decision object가 이유와 manual review 여부를 직렬화하게 하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 3개가 같은 target이면 ready, 수가 모자라면 이유가 나와야 하고, target 두 개면 무조건
> 막아야 해. high-risk도 기본은 검토 대기로 두되, 검토된 workflow에서는 Policy 옵션으로
> 풀 수 있게 해줘. 기존 ledger export나 Update DB도 깨지면 안 돼.

**작업 단계 — Red 테스트**

- 일관된 target cluster의 ready decision
- minimum record 부족 사유
- conflicting target과 high-risk의 manual review decision
- 명시적 high-risk override

```bash
python3 -m unittest tests.test_memory_ledger -q
# ModuleNotFoundError: brainwash.memory.promotion
```

Commit: `f71579e test(implement-06): specify memory promotion policy`

## Prompt 3 — 구조와 회귀 검증

**Developer**

> 기존 `promotion_report(many_min)` 쓰는 쪽은 그대로 두고, policy를 넘길 때만 예외 workflow를
> 선택할 수 있게 해줘. report 순서도 실행마다 바뀌지 않게 하고 전체 테스트까지 확인해줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 4 tests — OK

python3 -m unittest discover -s tests -q
# Ran 106 tests — OK
```

`PromotionPolicy`가 cluster를 size와 key 기준으로 안정적으로 정렬하고, 기존 convenience API는
default Policy로 위임하도록 구현했습니다.

Commit: `45b28e2 refactor(implement-06): separate memory promotion policy`
