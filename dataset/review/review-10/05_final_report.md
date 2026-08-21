# R-A10 routing lane 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 현황 및 이슈

`feat(review-10): split routing lanes`는 mixed correction batch를 lane별로 나누고 각 lane에 기존
router policy를 적용합니다. 최초 PR `46f99d5`은 `brainwash/router.py`에 56줄을 추가했고, 기존 전체
test 128건을 통과했습니다.

최초 구현은 non-behavior request를 하나의 memory lane에 넣었습니다. broad/domain correction이 섞이면
BroadScopeRoutingPolicy가 factual memory correction까지 training으로 route할 수 있었고, lane 결과에는
원래 input position도 없었습니다. routing은 작은 분류 실수가 다른 algorithm 선택으로 이어지는 경계이므로,
partition ownership·결과 재연결·기존 policy 보존을 코드리뷰 관점에서 확인하기 위해 이 활동을 선정했습니다.

## 2. 주요 검토 및 반영

### 2.1 broad/domain lane의 분리

초기 구현은 non-behavior request 전체를 memory lane에 넣었습니다. 그래서 broad domain correction 하나가 섞이면 BroadScopeRoutingPolicy가 factual correction까지 QLoRA training으로 route할 수 있었습니다. 반영 후에는 behavior, broad, memory bucket을 분리해 각 lane이 기존 policy가 기대하는 request 집합만 받도록 했습니다.

### 2.2 original request position 보존

lane은 policy presentation order로 반환되므로 input order와 다를 수 있습니다. [Python enumerate 공식 문서](https://docs.python.org/3/library/functions.html#enumerate)는 iterable을 index와 value 쌍으로 제공하므로, split 시점에 position을 함께 보관하는 데 직접 맞습니다.

    list(enumerate([fact, behavior]))
    # [(0, fact), (1, behavior)]

반영 후 RoutingLane은 request_positions를 JSON에도 포함합니다. 따라서 consumer는 lane metadata를 사용해 routing result를 원래 input report와 다시 연결할 수 있습니다.

## 3. 활동 내용

리뷰 의도는 기존 `RoutingPolicy` Strategy를 교체하지 않고, lane partition이 각 Policy가 기대하는 request
집합을 넘기도록 바로잡는 것이었습니다. 리뷰어는 fact request와 broad domain request가 같은 lane에 있을
때 factual correction이 QLoRA로 바뀌는 mixed input 사례를 제시하고, behavior·broad·memory 세 lane으로
분리하는 regression test를 요청했습니다.

또한 lane의 presentation order와 caller의 input order는 다른 정보라는 점을 구분했습니다. Python
`enumerate()` 문서를 근거로 split 시점에 position을 보존하는 방법을 제안했고, 작업자는 test commit에서
broad separation과 position contract를 먼저 실패시킨 뒤 fix commit에서 `request_positions`를
`RoutingLane`과 JSON output에 추가했습니다. style/safety 분류와 stable lane order도 스레드에서 확인해
Policy 의미를 임의의 kind 검사로 복제하지 않도록 했습니다.

리뷰 문화 측면에서는 routing 결과를 “어떤 lane에 속하는가”와 “원래 입력의 몇 번째인가”로 나누어
설명했습니다. 이로써 reviewer가 단순 결과값이 아니라 partition 규칙, policy ownership, consumer가
필요로 하는 metadata를 함께 검토할 수 있었습니다.

## 4. 기대 효과

broad/domain training 요구가 small factual correction의 Memory Store route를 바꾸지 않게 되어 algorithm
선택의 격리가 보장됩니다. `request_positions`가 남으므로 consumer는 lane presentation order와 무관하게
routing result를 원래 evaluation/report 행에 다시 연결할 수 있습니다.

팀은 routing 변경을 조건문 추가가 아니라 lane ownership과 output metadata contract의 변경으로 인식하게
됩니다. 이후 route partition 리뷰에서는 mixed input, 기존 Policy 적용 범위, input-order 복원 가능성을
공통 기준으로 확인할 수 있습니다. 이 기준은 새 lane을 추가할 때 기존 factual·behavioral 경로에
의도하지 않은 algorithm 변경이 생기는 것을 줄입니다.

## 5. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 46f99d5 | RoutingLane 및 lane batch API | 전체 128건 통과 |
| 리뷰 명세 | 33eaba4 | broad separation·request position test | 초기 구현에서 1 failure, 1 error 확인 |
| 리뷰 반영 | 88560d5 | three-lane partition과 position metadata | router 11건, 전체 130건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
