# I-A8 개발 활동 보고서 — routing Policy 분해

## 1. 현황 및 이슈

`BrainwashRouter.route()`는 behavior, broad/domain, fact, mixed batch의 판정 순서와 각
`AlgorithmPlan`의 warnings·stats·verifier field 생성을 한 메서드에서 수행했습니다. 새 route 조건을
읽으려면 앞선 모든 분기와 공통 payload 조립 코드를 함께 따라가야 했습니다.

가독성 측면에서는 “어떤 조건이 route를 소유하는지”와 “선택된 plan을 어떻게 직렬화하는지”가 섞여
precedence를 한눈에 확인하기 어려웠습니다. 유지보수성 측면에서는 lane을 추가할 때 공통 field를
누락하거나 기존 분기보다 잘못된 위치에 조건을 삽입할 위험이 있었습니다. route 선택 규칙과 plan
생성을 독립된 책임으로 나누고, precedence를 명시적인 순서로 표현할 필요가 있었습니다.

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

## 3. 활동 내용

구현 의도는 기존 if/elif precedence를 Chain of Responsibility 형태의 Policy sequence로 드러내고,
공통 `AlgorithmPlan` 생성은 `RoutingContext`의 Factory에 모으는 것이었습니다. 먼저 존재하지 않는
routing Policy를 사용하는 Red 테스트를 작성해 default sequence, injected Policy의 shared batch stats,
abstract selection contract를 고정했습니다.

`BehaviorRoutingPolicy`, `BroadScopeRoutingPolicy`, `FactRoutingPolicy`는 자신이 담당하는 batch에서만
plan을 반환하고, 해당하지 않으면 `None`으로 다음 Policy에 위임합니다. `MixedRoutingPolicy`는 terminal
fallback을 담당합니다. `default_routing_policies()`가 이 순서를 명시하고 `BrainwashRouter`는 첫 plan만
채택합니다.

`RoutingContext.plan()`은 warnings, thresholds, stats, verifier requirement 등 공통 field를 한 곳에서
채우므로 각 Policy는 algorithm·reason·scale·control mode만 결정합니다. custom sequence가 모두
defer하면 `RuntimeError`, empty batch는 기존과 같은 `ValueError`로 처리해 잘못된 구성도 조기에
드러나게 했습니다. 구현 후 다음 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_router -q
# Ran 9 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 8 tests — OK

python3 -m unittest discover -s tests -q
# Ran 113 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 기대 효과

route precedence가 class sequence로 보이므로 새 팀원이 긴 조건문을 해석하지 않아도 전체 결정 순서를
확인할 수 있습니다. 새로운 lane은 Policy 하나와 위치를 추가하는 방식으로 확장할 수 있고, 공통 plan
field는 Factory가 보장하므로 serialization drift와 field 누락 가능성이 줄어듭니다.

리뷰 과정에서는 routing을 “Policy가 소유권을 판단하고 Context가 공통 결과를 만든다”는 기준으로
정리했습니다. 팀원은 route 변경을 조건 추가가 아니라 precedence·ownership·terminal fallback의
문제로 논의할 수 있습니다. 이 공통 인식은 custom Policy 리뷰에서도 default path와 설정 오류를
구분하게 하며, 특정 lane 수정이 다른 lane의 payload까지 바꾸는 일을 예방합니다.

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
