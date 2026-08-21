# I-A8 개발 활동 보고서 — routing Policy 분해

## 1. 배경

`BrainwashRouter.route()`는 behavior, broad/domain, fact, mixed batch의 route 선택과 각
`AlgorithmPlan` 공통 field 생성을 한 메서드에서 수행했습니다. 새 lane을 추가하거나 precedence를
검토할 때 분기와 warnings·stats·verifier field를 함께 추적해야 했습니다.

이번 변경에서는 lane별 선택을 `RoutingPolicy` Strategy로 분리하고, `RoutingContext`가 immutable
batch facts와 common plan construction을 맡게 했습니다. Router는 precedence orchestration만 수행하며
기존 algorithm/control mode/fallback 결과는 유지합니다.

## 2. Commit 및 PR 경계

- base: `main` / `5b7ce2f912960aeac1a2e65136e4b9c298931af5`
- Red 테스트: `d2805626f4c2831914d95523815e452ca7032f08`
  `test(implement-08): specify routing policy precedence`
- 최초 PR 및 최종 head: `f590e3d52352574742b686886b7f10d8cebcb306`
  `refactor(implement-08): decompose routing policies`
- 최종 `main`: `f590e3d52352574742b686886b7f10d8cebcb306`

최초 head에서 Policy precedence, common `AlgorithmPlan` field 책임, custom sequence의 invalid
configuration 보호를 검토했습니다. 코드 결함은 발견되지 않아 Change Request나 후속 commit은 만들지
않았습니다.

## 3. TDD 및 동작 보존 검증

Red 테스트는 존재하지 않는 routing Policy import에서 실패했습니다. default sequence, injected
Policy의 shared batch stats, abstract selection contract를 먼저 명세로 고정했습니다.

구현 후 아래 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_router -q
# Ran 9 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 113 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 구조 개선

`RoutingPolicy`는 한 batch를 소유하면 plan을 반환하고, 소유하지 않으면 `None`으로 다음 lane에
위임하는 Strategy입니다. `BehaviorRoutingPolicy`, `BroadScopeRoutingPolicy`, `FactRoutingPolicy`가
focused lane을 처리하고 `MixedRoutingPolicy`가 terminal fallback을 담당합니다.

`RoutingContext`는 `BatchStats`, configured thresholds, warnings, stats payload를 한 번 만들고
`plan()` Factory를 통해 common `AlgorithmPlan` fields를 채웁니다. 따라서 Policy는 algorithm·reason·
scale·control mode처럼 lane-specific 값만 결정하며, serialization contract가 분기마다 복제되지 않습니다.

`BrainwashRouter`는 default Policy tuple 또는 caller가 준 custom sequence를 순회합니다. empty batch는
기존처럼 `ValueError`로 거절하고, terminal Policy가 없는 custom sequence는 `RuntimeError`로 드러나게
했습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 326줄 |
| 삭제 | 144줄 |
| 합계 | 470줄 |
| 파일 | 2개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/router.py`, `tests/test_router.py`입니다. 300줄을 넘었지만 네 routing lane의
Chain of Responsibility, common plan Factory, custom Policy injection, precedence regression test를
하나의 coherent refactor로 완료한 결과입니다.

## 6. 리뷰 결과

리뷰에서는 behavior → broad/domain → fact → mixed precedence, context가 common plan field를 만드는
경계, custom Policy와 missing terminal fallback의 책임을 확인했습니다. 세 material thread와 Router·
Update DB·전체 suite 검증을 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
