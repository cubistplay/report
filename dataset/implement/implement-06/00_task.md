# Implement-06 — memory promotion Policy 분리

## 개발 요청

Memory Ledger의 active record를 training cluster로 올릴 수 있는지 판단하는 기준을 정리해 주세요.
현재는 record 수만 `many_min` 이상이면 promotion 가능으로 표시해, target 충돌이나 high-risk
record가 있어도 자동 training 후보처럼 보입니다.

## 완료 조건

- 최소 record 수, target 일관성, high-risk 검토 요구를 독립 규칙으로 표현합니다.
- ledger는 active record 수집과 artifact export를 맡고, promotion 판단은 별도 Policy가 맡습니다.
- report에는 ready 여부, target 목록, 이유, manual review 필요 여부를 남깁니다.
- 명시적 Policy를 주면 high-risk 검토 규칙을 해제할 수 있습니다.
- 기존 ledger artifact export와 Update DB ingestion을 회귀 테스트로 확인합니다.

## 구현 제약

- 테스트를 먼저 작성하고 `test → refactor` commit 순서를 유지합니다.
- 변경은 `brainwash/memory/ledger.py`, `brainwash/memory/promotion.py`,
  `tests/test_memory_ledger.py`에 한정합니다.
- 기존 `promotion_report(many_min)` 호출 형식은 유지합니다.

## 작업 단위

이번 변경은 training promotion이라는 하나의 정책 경계를 `PromotionPolicy`와 세 Rule로
분리하는 구조 개선입니다. 300줄을 넘지만 Rule·Decision Value Object·ledger 위임·정책별
회귀 테스트가 함께 필요한 하나의 reviewable 단위입니다.
