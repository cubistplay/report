# feat: export memory snapshots

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

MemoryLedger artifact export에 as_of 경계를 추가하고, snapshot의 active/inactive totals와 inactive reason을 기록하는 memory_snapshot.json을 생성합니다.

## 주요 변경사항

- MemorySnapshotManifest Value Object를 추가했습니다.
- active snapshot과 lifecycle audit export가 선택적 as_of를 받습니다.
- memory_snapshot.json이 export 시점, active/inactive count, inactive reason을 기록합니다.
- 기존 artifact path return mapping에 memory_snapshot을 추가했습니다.

## 설계 의도

Snapshot Manifest 패턴으로 artifact bundle의 시간 경계와 요약 통계를 명시했습니다. ledger 원본은 audit 재현을 위해 그대로 남기고, active memory와 lifecycle decision은 snapshot view로 분리합니다.

~~~mermaid
flowchart LR
  L[MemoryLedger] --> A[active_memory]
  L --> D[memory_lifecycle]
~~~

~~~mermaid
flowchart LR
  L[MemoryLedger] --> S[MemorySnapshotManifest]
  S --> A[active_memory]
  S --> D[memory_lifecycle]
  S --> I[memory_index]
  S --> P[promotion_report]
~~~

## Review Points

1. **snapshot boundary** — as_of가 bundle의 모든 derived artifact에 동일하게 적용되는지 검토 부탁드립니다.
2. **timezone contract** — public export API가 naive datetime을 어떻게 해석해야 하는지 확인 부탁드립니다.

## PR Type

- [x] ✨ Feature
- [ ] 🐛 Bugfix
- [ ] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

    python3 -m unittest discover -s tests -q

기존 전체 test 124건이 통과했습니다. 최초 PR에는 과거 as_of에서 모든 파생 artifact가 같은 결과를 내는지와 naive datetime 입력을 직접 검증하는 test는 포함하지 않았습니다.

## Formatting

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최초 PR snapshot을 포함한 최종 변경 파일은 `black --check`를 통과했으며, 재작성 전후 변경 Python 파일의 AST도 동일하게 확인했습니다.

## Todos

- [ ] 리뷰 의견 반영
