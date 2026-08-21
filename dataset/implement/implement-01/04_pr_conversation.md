# I-A1 PR 대화 — locality 평가 구조 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `0072332` · Change Request 없음

## 스레드 1 — locality 비교에서 정규화하는 범위

**위치** `brainwash/eval/metrics.py` (`outputs_equivalent`)

**리뷰어 · 질문 🔎**

locality는 target answer를 맞혔는지가 아니라 baseline 출력과 같은지를 보네요.
비교 전에 공백과 대소문자를 정규화하는 이유가 있나요? 원문이 달라졌다는 신호를
너무 많이 놓치지는 않을지 궁금합니다.

**작업자 · 답변 💬**

현재 evaluator는 deterministic text output을 대상으로 합니다. 줄바꿈이나 대소문자만
다른 경우를 regression으로 세면 생성 형식 차이 때문에 locality가 흔들립니다. 그래서
공백과 대소문자만 무시하고, 단어가 달라지는 변화는 그대로 실패하게 했습니다.

**리뷰어 · 후속 질문 💭**

그렇다면 semantic equivalence까지 넓히지 않은 것도 의도인가요? 같은 뜻의 문장이라도
표현이 다르면 지금은 실패합니다.

**작업자 · 답변**

네. semantic 비교는 matcher 설정과 threshold에 영향을 받습니다. 이번 계약은 baseline
보존을 재현 가능하게 확인하는 것이어서 deterministic 정규화까지만 적용했습니다.
semantic locality 정책이 필요해지면 별도 evaluator로 추가할 수 있습니다.

**리뷰어 · 확인 ✅**

비교 범위를 좁게 유지한 이유가 명확합니다. 현재 `normalize_text`와 regression 테스트가
그 경계를 충분히 보여 줍니다.

## 스레드 2 — baseline이 없을 때의 점수 의미

**위치** `brainwash/eval/harness.py` (`LocalityPreservationEvaluator.evaluate`)

**리뷰어 · 질문 🔎**

baseline generator가 없으면 locality case를 실패가 아니라 `unscored`로 처리합니다.
그 경우 전체 `passed`와 locality 점수에는 어떻게 반영되는지 확인하고 싶습니다.

**작업자 · 답변 💬**

baseline이 없으면 수정 전 출력과 비교할 근거가 없으므로 실패라고 할 수 없습니다.
`passed`에는 넣지 않고 `unscored`를 올립니다. kind별 점수도 scored case만 분모로 써서
모든 locality case가 미평가면 `None`이 됩니다.

**리뷰어 · 후속 질문 💭**

그렇게 하면 호출자가 `locality == 0.0`과 `locality is None`을 구별할 수 있겠네요.
보고서 근거에도 왜 미평가인지 남나요?

**작업자 · 답변**

네. `unscored_cases`에는 case id, prompt, output, 그리고
`baseline_generator_required` 사유가 남습니다. 실패 목록과 분리했기 때문에
평가 불가와 실제 regression을 같은 상태로 보지 않습니다.

**리뷰어 · 확인 👍**

점수와 근거의 의미가 일관됩니다. missing baseline 테스트가 `None` 점수와 사유까지
확인하므로 이 정책은 고정되어 있습니다.

## 스레드 3 — evaluator 입력 경계

**위치** `brainwash/eval/harness.py` (`EvaluationContext`)

**리뷰어 · 질문 🔎**

`CaseEvaluator`가 case, 출력, baseline 출력을 각각 받지 않고 `EvaluationContext`를
받도록 바꿨네요. 지금은 입력이 세 개뿐인데 context를 따로 둔 이유가 있나요?

**작업자 · 답변 💬**

evaluator마다 필요한 입력이 늘 때마다 Protocol method를 바꾸고 싶지 않았습니다.
Harness가 context를 한 번 만들고 evaluator는 그 값만 읽습니다. 따라서 사용자 지정
evaluator도 Harness 내부 상태나 generator를 알 필요가 없습니다.

**리뷰어 · 후속 질문 💭**

context에 generator 자체를 넣지 않은 것은 evaluator가 재생성하지 못하게 하려는
의도인가요?

**작업자 · 답변**

맞습니다. 생성은 Harness의 실행 책임으로 제한했습니다. evaluator는 이미 생성된 출력과
평가 입력만 받아 판정하므로 evaluator별로 호출 횟수나 순서가 달라지지 않습니다.

**리뷰어 · 확인 📌**

사용자 지정 evaluator 테스트가 context의 output을 직접 확인하고 있어, 실행과 판정의
경계가 코드와 테스트에 같이 드러납니다.

## 스레드 4 — registry 검증 시점

**위치** `brainwash/eval/harness.py` (`EvaluatorRegistry.validate`)

**리뷰어 · 질문 🔎**

사용자 지정 registry에 필요한 kind가 빠진 경우, 오류가 나더라도 앞선 사례 몇 개는
이미 generate된 뒤일 수 있지 않을까 확인했습니다. 배치 평가에서 부분 실행이 남으면
곤란할 수 있어서요.

**작업자 · 답변 💬**

반복문에 들어가기 전에 모든 case kind와 registry key의 차집합을 계산합니다. 하나라도
빠져 있으면 generator는 한 번도 호출되지 않습니다. `generator.prompts == []` assertion으로
이 순서를 테스트에 고정했습니다.

**리뷰어 · 후속 질문 💭**

기본 registry와 빈 사용자 registry를 구분하는 방식도 같은 정책인가요?

**작업자 · 답변**

기본값은 `evaluators is None`일 때만 사용합니다. 빈 mapping은 호출자가 준 registry로
보존하고 검증에서 누락 오류를 냅니다. 의도한 사용자 설정을 기본값으로 덮어쓰지 않기
위한 구분입니다.

**리뷰어 · 확인 ✅**

부분 실행을 막고 빈 설정도 숨기지 않는 순서가 적절합니다.

## 스레드 5 — 보고서 조립 책임

**위치** `brainwash/eval/harness.py` (`EvaluationReportBuilder`)

**리뷰어 · 질문 🔎**

Harness가 verdict를 바로 report dict로 만들지 않고 `EvaluationReportBuilder`에 넘기는
구조가 늘어난 것처럼 보입니다. 이 분리를 유지할 이유가 있을까요?

**작업자 · 답변 💬**

kind별 분모, 실패 근거, 미평가 근거는 같은 결과 규칙입니다. 이를 builder에 모으면
Harness에는 generator 실행과 evaluator 선택만 남습니다. 결과 필드가 추가돼도 실행
반복문을 바꾸지 않아도 됩니다.

**리뷰어 · 확인 ✅**

locality와 `unscored`가 추가되면서 결과 조립 자체가 독립된 책임이 됐습니다. 과도한
분리가 아니라는 점을 확인했습니다.

## 승인

**리뷰어 · Approve ✅**

baseline 비교 범위, 미평가 점수 정책, evaluator 실행 경계, 사전 registry 검증, 결과
조립 책임이 각각 분명합니다. baseline regression과 missing baseline, 사용자 지정
evaluator, 사전 검증, behavior case가 테스트로 확인되어 승인합니다.
