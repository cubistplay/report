# R-A10 PR 대화 — routing lane 검토

> Synthetic GitHub artifact: true  
> 최초 검토 branch: review/review-10-routing-lanes (175520e) · 최종 head: 69869c9

## 스레드 1 — broad/domain correction이 factual memory lane을 training으로 바꿈

**위치** brainwash/router.py (route_lanes, 초기 PR 327-340행)  
**심각도** blocking · **Change Request**

**리뷰어 · 댓글 :thinking:**

현재 non-behavior request를 전부 memory lane에 넣습니다. domain 또는 broad request가 fact patch와 같이 들어가면 BroadScopeRoutingPolicy가 memory lane 전체를 QLORA_SFT로 선택합니다. 그러면 작은 factual correction도 MEMORY_STORE 대신 training route로 바뀝니다.

    fact request            -> MEMORY_STORE가 기대됨
    broad domain request    -> QLORA_SFT가 기대됨
    현재 memory lane        -> BroadScopeRoutingPolicy가 전체를 QLORA_SFT

domain kind나 broad/domain/global scope를 별도 broad lane으로 분리해 주세요. fact lane은 Memory Store plan, broad lane은 QLoRA plan이 나오는 mixed input test도 부탁드립니다. :mag:

**작업자 · 답변 :speech_balloon:**

맞습니다. behavior 여부만으로 나눈 것은 너무 거친 분류였습니다. domain/broad는 사실상 memory patch와 다른 routing lane인데 기존 broad policy에 맡긴다는 점만 보고 같은 bucket에 넣었습니다. broad lane을 추가하고 factual memory lane과 분리하겠습니다.

**리뷰어 · 후속 :+1:**

좋습니다. router가 이미 broad scope를 독립 policy로 취급하므로 lane split도 같은 ownership을 따라가면 됩니다. 신규 policy를 만들 필요 없이 partition만 바로잡는 범위라 적절합니다.

**작업자 · 반영 :white_check_mark:**

69869c9 fix(review-10): preserve routing lane boundaries에서 behavior, broad, memory 세 lane으로 분리했습니다. fact와 broad domain을 같이 넣어도 memory plan은 MEMORY_STORE, broad plan은 QLORA_SFT인지 regression test로 고정했습니다.

**리뷰어 · 확인 :tada:**

확인했습니다. domain training requirement가 factual memory correction의 routing을 끌어올리지 않습니다.

## 스레드 2 — lane split 뒤 original input position을 잃음

**위치** brainwash/router.py (RoutingLane, 초기 PR 105-122행)  
**심각도** important · **Change Request**

**리뷰어 · 댓글 :eyes:**

lane은 behavior → memory 순서로 반환되고 request_ids만 남깁니다. caller가 routing result를 input table이나 evaluation row에 다시 붙이려면 원래 위치를 알아야 합니다. ID는 stable key일 수 있지만 input order 자체를 복원하는 정보는 아닙니다.

[Python enumerate 공식 문서](https://docs.python.org/3/library/functions.html#enumerate)는 iterable을 index와 value 쌍으로 순회하는 기능을 설명합니다. split 직전에 index를 같이 보관하면 metadata는 간단하게 유지됩니다.

    list(enumerate([fact, behavior]))
    # [(0, fact), (1, behavior)]

RoutingLane에 request_positions를 추가하고, fact가 0번·behavior가 1번이던 mixed input에서 각 lane이 그 position을 보존하는 test를 추가해 주세요. :test_tube:

**작업자 · 답변 :memo:**

동의합니다. ID를 가진 request만 온다는 가정에 기대면 order-sensitive consumer가 불편해집니다. enumerate로 position과 request를 같이 bucket에 넣고 RoutingLane에 tuple로 저장하겠습니다.

**리뷰어 · 후속 :bulb:**

네. lane 결과 자체의 순서가 policy order여도 positions가 있으면 caller가 별도의 lookup 규칙 없이 original order report를 재구성할 수 있습니다. to_dict에도 같이 노출해 주세요.

**작업자 · 반영 :+1:**

같은 69869c9에서 request_positions를 RoutingLane과 JSON output에 추가했습니다. fact 0번, behavior 1번 input test가 각각 memory=(0,), behavior=(1,)을 확인합니다.

**리뷰어 · 확인 :white_check_mark:**

확인했습니다. policy lane order와 caller input order를 모두 보존할 수 있게 됐습니다.

## 스레드 3 — empty input 계약

**위치** BrainwashRouter.route_lanes  
**심각도** question

**리뷰어 · 질문 :thinking:**

route_lanes([])는 빈 RoutingLaneBatch를 반환하지 않고 route()와 같은 ValueError입니다. batch API라면 빈 결과도 가능해 보이는데, 기존 계약을 유지한 이유가 있나요?

**작업자 · 답변 :speech_balloon:**

route_lanes는 실행 가능한 routing plan을 만드는 API라 빈 input을 성공 결과로 만들면 caller가 execution stage까지 빈 batch를 넘길 수 있습니다. 기존 route()와 같은 오류로 처리해 upstream validation을 유지했습니다.

**리뷰어 · 확인 :+1:**

single route와 lane route의 invalid-input boundary가 같아 caller가 두 API를 다르게 처리할 필요가 없습니다.

## 스레드 4 — style/safety의 behavioral lane 분류

**위치** _lane_name  
**심각도** question

**리뷰어 · 질문 :shield:**

style과 safety kind도 behavior lane으로 갑니다. kind enum만 검사하지 않고 request.is_behavioral을 사용한 이유가 explicit chosen/rejected pair 때문인가요?

**작업자 · 답변 :memo:**

네. is_behavioral은 behavior/style/safety kind와 explicit preference pair를 함께 표현합니다. factual kind라도 chosen/rejected pair가 있으면 preference data로 route해야 하므로 lane partition도 이 property를 재사용했습니다.

**리뷰어 · 확인 :white_check_mark:**

schema의 semantic property를 재사용해 routing policy와 lane partition이 다른 행동 정의를 갖지 않게 됐습니다.

## 스레드 5 — lane order의 의미

**위치** route_lanes의 lane_requests 순서  
**심각도** question

**리뷰어 · 질문 :eyes:**

결과 lane 순서는 behavior, broad, memory입니다. 이 순서가 input order가 아니라 stable presentation order라는 점을 consumer가 알아야 할 것 같습니다. 별도 sort key를 추가할 필요는 없나요?

**작업자 · 답변 :speech_balloon:**

이 순서는 processing priority가 아니라 stable presentation order입니다. consumer가 input order가 필요하면 request_positions를 사용합니다. lane name 자체가 stable key라 별도 numeric sort key까지 추가하면 public contract만 넓어진다고 판단했습니다.

**리뷰어 · 확인 :+1:**

lane order와 original order의 역할이 request_positions로 구분되므로 현재 metadata면 충분합니다.

## 최종 승인

**리뷰어 · Approve :white_check_mark:**

broad/domain과 factual memory correction을 독립 lane으로 분리하고, 각 lane에 original input position을 남겼습니다. existing policy Strategy는 유지하면서 partition 책임만 보완했습니다. tests.test_router 11건과 전체 test 130건 통과를 확인하여 승인합니다.
