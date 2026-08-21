# feat: list promotion candidates

> Synthetic GitHub artifact: true  
> 최초 PR 시점의 설명입니다. 이후 리뷰 대화와 반영 commit은 포함하지 않습니다.

## 요약

Update DB의 active memory 중 promotion evidence threshold를 충족한 그룹을 PromotionCandidate로 반환합니다. 후보는 subject, relation, scope key와 record IDs, targets, high-risk signal을 함께 제공해 training promotion 전에 검토할 수 있습니다.

## 주요 변경사항

- 불변 PromotionCandidate Value Object를 추가했습니다.
- promotion_candidates(many_min)가 active memory를 key별로 집계합니다.
- 후보는 high-risk record 또는 복수 target이 있으면 manual review가 필요하다고 표시합니다.
- 후보가 없거나 threshold가 유효하지 않은 경우의 경계를 API에서 명시합니다.

## 설계 의도

후보 조회와 실제 training promotion을 분리했습니다. PromotionCandidate는 조회 시점의 사실을 표현하는 Value Object이고, training cluster를 생성하거나 policy exception을 승인하는 책임은 호출자에 남깁니다. 이는 Query Object 패턴을 적용해 read model을 명시적으로 만드는 구조입니다.

~~~mermaid
flowchart LR
  M[(memory_records)] --> T[training decision]
~~~

~~~mermaid
flowchart LR
  M[(active memory_records)] --> Q[promotion_candidates]
  Q --> C[PromotionCandidate]
  C --> R[manual review]
  C --> T[training decision]
~~~

## Review Points

1. **candidate determinism** — audit/export consumer가 반복 호출해도 같은 record/target 순서를 받는 경계가 적절한지 검토 부탁드립니다.
2. **policy alignment** — candidate의 conflict signal과 기존 PromotionPolicy의 target 비교 규칙이 일치하는지 확인 부탁드립니다.

## PR Type

- [x] ✨ Feature
- [ ] 🐛 Bugfix
- [ ] ♻️ Refactoring (no functional changes, no api changes)
- [ ] 🎨 Code style update
- [ ] 📚 Docs
- [ ] 🔧 Other

## 테스트

    python3 -m unittest discover -s tests -q

기존 전체 test 122건이 통과했습니다. 최초 PR에는 candidate result의 순서와 target 정규화를 직접 검증하는 test는 포함하지 않았습니다.

## Formatting

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 최초 PR snapshot을 포함한 최종 변경 파일은 `black --check`를 통과했으며, 재작성 전후 변경 Python 파일의 AST도 동일하게 확인했습니다.

## Todos

- [ ] 리뷰 의견 반영
