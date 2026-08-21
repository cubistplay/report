# I-A4 PR 대화 — dynamic evidence action 단계 분리

> Synthetic GitHub artifact: true  
> 최초 검토 head: `509a333` · Change Request 없음

## 스레드 1 — action 종류와 실행 책임

**위치** `brainwash/decomposition.py` (`DynamicActionStageRegistry`)

**리뷰어 · 질문 🔎**

기존 loop의 `if action.action == ...` 분기를 stage registry로 옮겼네요. `answer`와
`ask`만 Strategy로 분리하고 나머지는 `InvalidActionStage`로 보내는 이유가 있나요?

**작업자 · 답변 💬**

현재 planner protocol에서 실행 가능한 action은 `answer`와 `ask`뿐입니다. 다른 값은
새 action으로 추측해서 실행하기보다, action log를 남긴 뒤 기존과 같은 실패 결과로
끝내는 편이 안전합니다.

**리뷰어 · 후속 질문 💭**

그렇다면 invalid action이 관찰 기록 없이 실패하는 일은 없나요?

**작업자 · 답변**

없습니다. executor가 stage를 선택하기 전에 `state.record_action()`을 호출합니다.
새 integration test도 `give_up` action이 plan의 첫 action으로 남은 상태에서
`dynamic_failed`가 되는 것을 확인합니다.

**리뷰어 · 확인 ✅**

실행 가능한 action의 범위는 좁게 유지하면서 planner의 잘못된 출력을 분석할 근거도
보존하는 구조입니다.

## 스레드 2 — state가 소유하는 기록의 일관성

**위치** `brainwash/decomposition.py` (`DynamicEvidenceState`)

**리뷰어 · 질문 🔎**

step 결과, `answers_by_id`, memory dependency, memory 사용 여부를 state에 모았습니다.
이전에는 서로 다른 분기에서 각각 갱신했는데, state가 너무 많은 책임을 갖지는 않나요?

**작업자 · 답변 💬**

이 값들은 한 번의 dynamic 실행에만 존재하고 함께 변합니다. step을 기록하면 다음 query
rendering에 쓸 answer도 생기고, memory dependency가 추가되면 결과 mode도 바뀝니다.
따라서 실행 단위의 일관된 상태로 묶는 편이 분기마다 따로 갱신하는 것보다 안전합니다.

**리뷰어 · 후속 질문 💭**

같은 memory candidate가 planner action과 recovery 양쪽에서 참조되면 dependency가 두 번
들어갈 수 있는데, 그 경우도 state가 막나요?

**작업자 · 답변**

네. `record_dependencies()`가 id 집합을 기준으로 처음 들어온 dependency만 추가합니다.
단위 테스트에서 같은 id를 두 번 전달해도 plan에는 한 항목만 남는지 확인했습니다.

**리뷰어 · 확인 👍**

서로 의존하는 실행 기록을 한 곳에서 갱신하고 중복을 막는 책임은 state에 두는 것이
자연스럽습니다.

## 스레드 3 — 결과 보존 범위

**위치** `DynamicEvidenceExecutor.answer`, `AnswerActionStage`, `AskActionStage`

**리뷰어 · 질문 🔎**

stage 분리 뒤에도 memory recovery, placeholder rendering, 조기 answer type 종료가 모두
같은 순서로 동작하는지 확인하고 싶습니다. 구조가 바뀌면서 경로 하나가 달라지기 쉬운
부분이라서요.

**작업자 · 답변 💬**

`AskActionStage`는 기존 ask 경로를 그대로 호출하고, recovery 뒤에는 다음 planner turn으로
넘깁니다. step 결과가 target answer type을 만족하면 같은 시점에 성공 결과를 만듭니다.
기존 decomposition 테스트 22건이 이 경로들을 계속 검증합니다.

**리뷰어 · 후속 질문 💭**

대표 사례만 통과한 것이 아니라 결과 payload 자체도 이전과 비교했나요?

**작업자 · 답변**

네. memory step 하나를 실행한 뒤 `answer_from="s1"`로 끝나는 scripted planner를
변경 전 head와 현재 head에서 실행해 `DecompositionAnswer.to_dict()` 전체 JSON을
비교했습니다. `diff` 출력은 없었습니다.

**리뷰어 · 확인 📌**

기존 경로 테스트와 전체 payload 동등성 비교가 함께 있어 구조 변경의 보존 범위를
확인할 수 있습니다.

## 승인

**리뷰어 · Approve ✅**

action dispatch, 실행 state, 결과 종료 책임이 분리되어 있으며 invalid action 기록과
dependency 중복 제거도 테스트로 고정되어 있습니다. 기존 dynamic 실행의 JSON 동등성까지
확인되어 승인합니다.
