# I-A8 PR 대화 — routing Policy 분해

> Synthetic GitHub artifact: true  
> 최초 검토 head: `f590e3d` · Change Request 없음

## 스레드 1 — routing precedence와 Chain of Responsibility

**위치** `brainwash/router.py` (`default_routing_policies`, `BrainwashRouter.route`)

**리뷰어 · 질문 🔎**

behavior, broad/domain, fact Policy가 `None`으로 다음 Policy에 위임하고 mixed Policy가 terminal
fallback입니다. 기존 if 순서가 바뀌지 않는다는 보장은 어디에 있나요?

**작업자 · 답변 💬**

`default_routing_policies()`가 behavior → broad/domain → fact → mixed 순서를 명시적으로 반환합니다.
Router는 이 tuple을 순서대로 순회해 첫 plan만 반환합니다. default class 순서를 직접 확인하는 test도
추가했고 기존 fact·behavior route test가 결과 계약을 계속 검증합니다.

**리뷰어 · 후속 질문 💭**

broad/domain은 fact ratio보다 먼저 처리되던 기존 동작도 그대로겠네요.

**작업자 · 답변**

맞습니다. `BroadScopeRoutingPolicy`는 `FactRoutingPolicy`보다 앞에 있어 domain kind나 broad scope가
있으면 QLoRA SFT를 먼저 선택합니다. mixed는 focused Policy가 모두 defer할 때만 도달합니다.

**리뷰어 · 확인 ✅**

precedence가 implicit if 순서가 아니라 named Policy sequence로 드러나 검토하기 쉬워졌습니다.

## 스레드 2 — context가 공통 plan field를 만드는 이유

**위치** `RoutingContext.plan`

**리뷰어 · 질문 🔎**

Policy별로 `AlgorithmPlan`을 직접 만들지 않고 context factory를 거치게 했습니다. algorithm과
reason은 각 lane에 있는데 warnings와 stats를 공통화한 이유가 있나요?

**작업자 · 답변 💬**

warnings, threshold payload, batch size, verifier requirement는 route를 선택한 뒤에도 batch 전체에
공통인 정보입니다. 각 Policy가 직접 만들면 새 lane에서 한 field를 빼거나 stats shape를 다르게 만들기
쉽습니다. `RoutingContext.plan()`이 공통 field를 채우고 Policy는 algorithm-specific 값만 결정합니다.

**리뷰어 · 후속 질문 💭**

custom Policy도 같은 context를 받으니 기존 plan serialization과 경고 contract를 재사용할 수 있겠네요.

**작업자 · 답변**

네. custom recording Policy test가 `context.stats.fact_count`를 확인한 뒤 Fact Policy에 위임합니다.
custom lane도 `context.plan()`을 사용하면 default lane과 같은 warnings·stats payload를 얻습니다.

**리뷰어 · 확인 👍**

공통 plan field가 단일 출처가 되어 lane 확장 시 serialization drift를 줄입니다.

## 스레드 3 — custom Policy injection과 empty batch 보호

**위치** `BrainwashRouter.__init__`, `RoutingPolicy`

**리뷰어 · 질문 🔎**

constructor가 custom Policy sequence를 받습니다. custom Policy가 아무 plan도 반환하지 않거나 empty
batch가 들어올 때 기존 error handling은 어떻게 되나요?

**작업자 · 답변 💬**

`route()`는 context를 만들기 전에 empty batch를 계속 `ValueError`로 거절합니다. 모든 injected Policy가
defer하면 `RuntimeError`를 내도록 해 terminal Policy를 빼는 설정 실수를 조기에 드러냅니다. abstract
`RoutingPolicy`는 `select()` 구현 없이 생성할 수 없게 했습니다.

**리뷰어 · 후속 질문 💭**

default 사용자는 이 RuntimeError를 보지 않겠네요. mixed Policy가 항상 마지막에 있으니까요.

**작업자 · 답변**

맞습니다. default tuple에는 terminal `MixedRoutingPolicy`가 포함됩니다. RuntimeError는 명시적으로
custom sequence를 구성한 caller가 fallback을 누락했을 때만 설정 오류를 알려 주는 경계입니다.

**리뷰어 · 확인 📌**

확장 지점을 열었지만 default path와 invalid configuration의 책임이 분명합니다.

## 승인

**리뷰어 · Approve ✅**

routing lane의 precedence와 terminal fallback을 보존하면서, context가 공통 `AlgorithmPlan` contract를
관리하도록 분리했습니다. custom Policy stats 전달과 기존 route 회귀 test를 확인하여 승인합니다.
