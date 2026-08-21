# I-A3 PR 대화 — algorithm registry 캡슐화

> Synthetic GitHub artifact: true  
> 최초 검토 head: `b6391f6` · 리뷰 반영 head: `f60cd2f`

## 스레드 1 — 빈 주입과 기본값의 구분

**위치** `brainwash/pipeline.py` (`BrainwashPipeline._resolve_registry`, 초기 PR 26-33행)

**리뷰어 · 질문 🔎**

기존 `registry or default_registry()`는 빈 dict도 기본값으로 바꿨는데, 이제 `None`만
기본값으로 처리합니다. 테스트에서도 빈 registry가 adapter 누락으로 실패하는데,
이 구분을 API 계약으로 잡은 이유가 있나요?

**작업자 · 답변 💬**

빈 registry는 테스트나 제한된 실행 환경에서 의도적으로 adapter를 끈 설정일 수 있습니다.
이를 기본값으로 덮어쓰면 호출자가 준 설정과 다른 adapter가 실행될 수 있어, `None`만
기본 registry 요청으로 해석했습니다.

**리뷰어 · 후속 질문 💭**

adapter가 없다는 것을 안 뒤에도 output directory가 생기면 호출자가 실패한 실행을
성공한 artifact처럼 오해할 수 있습니다. 확인 순서는 어떻게 바뀌었나요?

**작업자 · 답변**

`plan()` 뒤 `registry.require()`을 먼저 호출하고, adapter를 얻은 경우에만 `ensure_dir()`를
호출합니다. 빈 registry 테스트는 예외뿐 아니라 target path가 존재하지 않는지도 확인합니다.

**리뷰어 · 확인 ✅**

설정 오류를 숨기지 않고 파일 시스템 부작용도 막는 순서가 적절합니다.

## 스레드 2 — override에서 보존하는 값의 범위

**위치** `brainwash/pipeline.py` (`BrainwashPipeline.plan`, 초기 PR 35-45행)

**리뷰어 · 질문 🔎**

override에서 `AlgorithmPlan`을 새로 만들지 않고 `replace()`를 사용했네요. 지금은
algorithm과 reason만 다르지만, router가 만든 list나 stats 같은 값도 그대로 보존되는 것이
의도인가요?

**작업자 · 답변 💬**

의도한 범위입니다. 이 메서드의 계약은 사용자가 선택한 algorithm과 그 설명만 바꾸는
것이고, confidence·fallback·warnings 등 route 판단은 router 결과를 그대로 사용합니다.

**리뷰어 · 후속 질문 💭**

그 필드들이 mutable 값이어도 deep copy하지 않는 이유가 있나요?

**작업자 · 답변**

`AlgorithmPlan`은 frozen dataclass이고 pipeline은 반환 전후에 후보 목록이나 stats를
변경하지 않습니다. deep copy를 넣으면 값 보존이라는 목적과 관계없이 복사 비용과 의미를
추가하게 됩니다. 테스트는 override 결과가 `replace(routed, ...)`와 같은지 확인합니다.

**리뷰어 · 확인 👍**

선택 override가 route 판단을 다시 계산하지 않는다는 경계가 명확합니다.

## 스레드 3 — 중복 등록과 교체의 명시성

**위치** `brainwash/algorithms/registry.py` (`AlgorithmRegistry.register`, 초기 PR 43-48행)

**리뷰어 · 질문 🔎**

일반 dict처럼 같은 key를 넣으면 조용히 바꾸지 않고, `replace=True`일 때만 교체하게
했습니다. 왜 기본 동작을 거부로 정했나요?

**작업자 · 답변 💬**

같은 algorithm에 두 adapter가 등록되면 실행 대상이 바뀌는데, dictionary 대입은 그 사실을
남기지 않습니다. 등록 실수를 빨리 드러내고, 의도한 test double이나 adapter 교체만
호출부에서 명시하게 하려는 정책입니다.

**리뷰어 · 후속 질문 💭**

그러면 runtime plugin처럼 나중에 adapter를 바꾸는 경우도 막히지는 않나요?

**작업자 · 답변**

막지 않습니다. 그 경우에는 `replace=True`를 사용하면 됩니다. 새 테스트도 기본 중복은
오류이고, 명시적 교체 뒤 `require()`가 새 adapter를 돌려주는 것을 함께 확인합니다.

**리뷰어 · 확인 📌**

교체 기능은 유지하면서 실수로 덮어쓰는 경로만 막은 설계로 보입니다.

## 스레드 4 — mapping 호환성과 adapter 이름 계약

**위치** `brainwash/algorithms/registry.py` (`AlgorithmRegistry.from_mapping`, 초기 PR 30-41행)

**리뷰어 · 질문 🔎**

기존 dictionary 주입은 받되, key와 `adapter.name`이 다르면 registry 생성 시 오류를 냅니다.
이 검증을 실행 시점까지 미루지 않은 이유가 있나요?

**작업자 · 답변 💬**

registry key는 router가 선택하는 algorithm 이름이고, adapter 이름은 manifest와 artifact
이름에 사용됩니다. 둘이 다르면 route는 한 algorithm을 선택했는데 결과는 다른 이름으로
기록될 수 있습니다. 그래서 mapping을 registry로 바꾸는 경계에서 막았습니다.

**리뷰어 · 후속 질문 💭**

기본 registry의 `RagContextPatchAdapter`도 이번에 `RAG_CONTEXT_PATCH`를 명시해 만들었네요.
상속받은 기본 이름과 key가 다시 어긋나면 이 경계가 무너질 수 있습니다.

**리뷰어 · Change Request ❗**

RAG adapter identity를 직접 확인하는 regression test를 추가해 주세요. registry 생성이
성공한다는 사실만으로는 이후 생성자 변경에서 해당 이름이 계속 보존된다는 것을 명확히
보장하지 못합니다.

**작업자 · 답변**

반영하겠습니다. `default_registry().require(RAG_CONTEXT_PATCH).name`이 같은 enum인지
확인하는 테스트를 추가하겠습니다. 이 변경은 기존 RAG 경로에서 잘못 기록될 수 있던
adapter 이름을 바로잡는 초기 구현을 회귀로부터 보호합니다.

**작업자 · 리뷰 반영 commit**

`f60cd2f test(implement-03): cover rag adapter registration`

`test_default_registry_preserves_rag_adapter_identity`를 추가했습니다.

```bash
python3 -m unittest tests.test_pipeline_registry -q
# Ran 6 tests — OK

python3 -m unittest tests.test_pipeline -q
# Ran 5 tests — OK

python3 -m unittest discover -s tests -q
# Ran 83 tests — OK
```

**리뷰어 · 확인 ✅**

확인했습니다. RAG key·adapter name 불일치가 manifest와 artifact 이름으로 번지는 경로를
회귀 테스트가 막습니다. Change Request를 해결한 것으로 보입니다.

## 스레드 5 — Mapping 표면과 mutation 경계

**위치** `brainwash/algorithms/registry.py` (`AlgorithmRegistry`, 초기 PR 22-64행)

**리뷰어 · 질문 🔎**

`AlgorithmRegistry`가 `Mapping`을 구현합니다. 읽기 호출은 기존 `get()`과 반복을 그대로
쓸 수 있지만, 내부 dictionary를 외부에 노출하지 않는 이유가 있나요?

**작업자 · 답변 💬**

조회는 mapping 표면으로 호환시키고, mutation은 `register()` 하나로 제한하려는 의도입니다.
그래야 중복 등록과 교체 정책을 우회할 수 없습니다. pipeline은 registry의 등록 상태를
직접 수정하지 않고 `require()`만 사용합니다.

**리뷰어 · 확인 ✅**

호환이 필요한 읽기 API와 정책이 필요한 변경 API의 구분이 자연스럽습니다.

## 최종 승인

**리뷰어 · Approve ✅**

adapter 등록 정책은 `AlgorithmRegistry`로 모였고, pipeline은 계획 결정과 실행 순서만
담당합니다. 빈 주입, plan 보존, 명시적 교체, 이름 계약, 실패 전 부작용 방지 규칙을
확인했습니다. RAG identity regression test 반영까지 확인되어 승인합니다.
