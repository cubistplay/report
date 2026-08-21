# feat: batch memory trigger decisions

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

MemoryEditStore에 batch trigger API를 추가합니다. 여러 prompt의 memory trigger decision과 요청 수, triggered 수, blocked 수, reason summary를 한 결과 객체로 제공합니다.

## 주요 변경사항

- MemoryTriggerBatch Value Object를 추가했습니다.
- evaluate_many()가 prompt별 trigger decision을 묶습니다.
- retrieve_many()가 batch의 matched edit를 반환합니다.
- batch 결과는 JSON-compatible summary로 직렬화할 수 있습니다.

## 설계 의도

Batch Result 패턴으로 개별 MemoryTrigger와 evaluation summary의 책임을 분리했습니다. store는 matching과 safety gate를 계속 담당하고, MemoryTriggerBatch는 batch-level observability만 담당합니다.

~~~mermaid
flowchart LR
  P[prompt] --> S[MemoryEditStore]
  S --> T[MemoryTrigger]
~~~

~~~mermaid
flowchart LR
  P[prompt batch] --> S[MemoryEditStore]
  S --> B[MemoryTriggerBatch]
  B --> T[ordered trigger decisions]
  B --> M[summary metrics]
~~~

## Review Points

1. **input/result alignment** — duplicate prompt를 포함한 input batch가 결과와 한 항목씩 대응하는지 검토 부탁드립니다.
2. **metric semantics** — triggered_count와 blocked_count가 consumer가 읽는 operational 의미와 일치하는지 확인 부탁드립니다.

## PR Type

- [x] ✨ Feature
- [ ] 🐛 Bugfix
- [ ] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

    python3 -m unittest discover -s tests -q

기존 전체 test 126건이 통과했습니다. 최초 PR에는 duplicate prompt의 input cardinality와 no-match batch의 triggered count를 직접 검증하는 test는 포함하지 않았습니다.

## Todos

- [ ] 리뷰 의견 반영
