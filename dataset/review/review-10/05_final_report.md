# R-A10 routing lane 리뷰 결과 보고서

> Synthetic GitHub artifact: true

## 1. 검토 대상

feat(review-10): split routing lanes는 mixed correction batch를 lane별로 나누고 각 lane에 기존 router policy를 적용합니다. 최초 PR 46f99d5은 brainwash/router.py에 56줄을 추가했고, 기존 전체 test 128건을 통과했습니다.

## 2. 주요 검토 및 반영

### 2.1 broad/domain lane의 분리

초기 구현은 non-behavior request 전체를 memory lane에 넣었습니다. 그래서 broad domain correction 하나가 섞이면 BroadScopeRoutingPolicy가 factual correction까지 QLoRA training으로 route할 수 있었습니다. 반영 후에는 behavior, broad, memory bucket을 분리해 각 lane이 기존 policy가 기대하는 request 집합만 받도록 했습니다.

### 2.2 original request position 보존

lane은 policy presentation order로 반환되므로 input order와 다를 수 있습니다. [Python enumerate 공식 문서](https://docs.python.org/3/library/functions.html#enumerate)는 iterable을 index와 value 쌍으로 제공하므로, split 시점에 position을 함께 보관하는 데 직접 맞습니다.

    list(enumerate([fact, behavior]))
    # [(0, fact), (1, behavior)]

반영 후 RoutingLane은 request_positions를 JSON에도 포함합니다. 따라서 consumer는 lane metadata를 사용해 routing result를 원래 input report와 다시 연결할 수 있습니다.

## 3. 커밋 및 검증

| 단계 | Commit | 내용 | 검증 |
| --- | --- | --- | --- |
| 최초 PR | 46f99d5 | RoutingLane 및 lane batch API | 전체 128건 통과 |
| 리뷰 명세 | 33eaba4 | broad separation·request position test | 초기 구현에서 1 failure, 1 error 확인 |
| 리뷰 반영 | 88560d5 | three-lane partition과 position metadata | router 11건, 전체 130건 통과 |

최초 PR 이후에는 test commit과 fix commit을 순서대로 누적했고, rebase나 force push를 사용하지 않았습니다.

## 4. 결론

Composite Result와 기존 RoutingPolicy Strategy 구조는 유지했습니다. broad/domain training 요구가 small factual correction의 Memory Store route를 바꾸지 않게 됐으며, lane reordering도 request_positions로 추적할 수 있습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
