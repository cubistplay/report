# 프롬프트 히스토리 — I-A8

> 실제 대화 export가 아니라, 이 PR을 만들기 위한 가상 개발 프롬프트 시나리오입니다.
> 구현과 테스트 결과를 기준으로 작성했습니다.

## Prompt 1 — routing lane 분리

**Developer**

> router 한 군데에 behavior, domain, fact, mixed가 다 들어가 있어서 너무 길어. 지금
> precedence랑 결과는 유지하되 lane별 판단을 따로 빼고, 공통 plan 필드는 한 곳에서 만들게
> 정리해줘.

**Codex**

> `RoutingPolicy` Strategy를 behavior, broad/domain, fact, mixed lane으로 나누겠습니다.
> `RoutingContext`가 `BatchStats`, thresholds, warnings, common `AlgorithmPlan` 생성을 맡고,
> router는 순서대로 Policy를 실행해 첫 선택 결과만 반환하도록 하겠습니다.

## Prompt 2 — 테스트 우선 계약

**Developer**

> 기본 순서가 바뀌면 안 되고, 나중에 custom lane을 앞에 끼울 수 있어야 해. custom policy도
> 같은 batch stats를 받아야 하고 abstract policy는 구현 없이 못 쓰게 test로 남겨줘. 기존
> fact/behavior 결과도 그대로여야 해.

**작업 단계 — Red 테스트**

- default Policy class와 precedence
- injected Policy의 batch stats 수신 후 fact lane 위임
- abstract Policy의 selection contract

```bash
python3 -m unittest tests.test_router -q
# ImportError: BehaviorRoutingPolicy 등이 아직 없습니다.
```

Commit: `e5b458d test(implement-08): specify routing policy precedence`

## Prompt 3 — route 결과 보존

**Developer**

> small/middle/large fact, behavior pair, conflict memory mode, domain, mixed를 기존과 같은
> plan으로 만들어야 해. router 쪽만 건드리고 Update DB와 전체 test도 같이 확인해줘.

**작업 단계 — Green 검증**

```bash
python3 -m unittest tests.test_router -q
# Ran 9 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 113 tests — OK
```

Policy는 `None`으로 다음 lane에 위임하고, 마지막 `MixedRoutingPolicy`가 terminal fallback을
제공하도록 구현했습니다. full suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나
실패는 없었습니다.

Commit: `276f088 refactor(implement-08): decompose routing policies`
