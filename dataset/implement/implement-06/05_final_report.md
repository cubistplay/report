# I-A6 개발 활동 보고서 — memory promotion Policy 분리

## 1. 현황 및 이슈

Memory Ledger의 `promotion_report()`는 active record 수집, cluster grouping, training 가능 여부 판단,
report 직렬화를 한 메서드에서 처리했습니다. 기존 판정은 record 수가 `many_min` 이상인지에 집중해
target 충돌이나 high-risk record가 포함된 cluster도 training candidate처럼 보일 수 있었습니다.

가독성 측면에서는 boolean 결과만으로 “자료가 부족해서 보류됐는지”, “target이 충돌했는지”,
“사람의 승인이 필요한지”를 구분하기 어려웠습니다. 유지보수성 측면에서는 promotion 기준 하나를
추가할 때 persistence 책임을 가진 Ledger와 report 형식까지 함께 수정해야 했습니다. 정책 변경이
저장·조회 코드의 회귀로 이어질 수 있는 구조였기 때문에, 판단 규칙과 record 관리 책임을 분리할
필요가 있었습니다.

## 2. Commit 및 PR 경계

- base: `main` / `6536ec48b18442604972f6b5e83aba8ea53f7684`
- Red 테스트: `be806c97c136c4d61c6796d9f675ccbf168262d4`
  `test(implement-06): specify memory promotion policy`
- 최초 PR 및 최종 head: `2749f66ecb6830e1d77228e8dc34d75718bd8d3b`
  `refactor(implement-06): separate memory promotion policy`
- 최종 `main`: `2749f66ecb6830e1d77228e8dc34d75718bd8d3b`

최초 head에서 책임 경계, high-risk override의 안전성, 기존 ledger/Update DB 계약을 검토했습니다.
코드 결함은 발견되지 않아 Change Request나 후속 commit은 만들지 않았습니다.

## 3. 활동 내용

구현 의도는 Ledger를 persistence 경계로 유지하고, training eligibility를 이름이 드러나는 정책 객체로
옮기는 것이었습니다. 먼저 아직 존재하지 않는 `PromotionPolicy`를 사용하는 Red 테스트를 작성해
일관된 target, 최소 record 부족, conflicting target, high-risk review, reviewed workflow override의
다섯 계약을 고정했습니다.

`PromotionRule` Strategy와 `MinimumRecordsRule`, `ConsistentTargetRule`, `HighRiskReviewRule`을 도입해
각 차단 사유를 독립된 이름과 구현으로 표현했습니다. `PromotionPolicy`는 Rule finding을 조합하고,
`PromotionDecision`은 record ID·target·reason·manual review 여부를 함께 직렬화합니다.
`MemoryLedger.promotion_report(many_min)`는 기존 호출 형식을 유지하면서 default Policy에 active record를
전달하도록 단순화했습니다. reviewed workflow도 `require_risk_review=False`를 명시한 Policy에서만
허용해 hidden branch를 만들지 않았습니다.

이 구조는 긴 조건문을 “최소 근거”, “target 일관성”, “risk 검토”라는 읽을 수 있는 단위로 바꾸고,
새 gate를 추가할 때 Ledger가 아니라 Rule만 확장할 수 있게 합니다. 구현 후 다음 검증을 완료했습니다.

```bash
python3 -m unittest tests.test_memory_ledger -q
# Ran 7 tests — OK

python3 -m unittest tests.test_update_db -q
# Ran 4 tests — OK

python3 -m unittest discover -s tests -q
# Ran 106 tests — OK
```

전체 suite는 기존 sqlite connection `ResourceWarning` 2건을 출력했으나 test 실패는 없었습니다.

## 4. 기대 효과

promotion 기준이 Rule 단위로 분리되어 새로운 품질 gate를 추가하거나 기존 기준을 변경할 때의 수정
범위가 작아집니다. report에는 boolean뿐 아니라 차단 reason과 manual review 여부가 남으므로 운영자와
리뷰어가 결과를 다시 코드로 역추적하지 않고도 판단 근거를 이해할 수 있습니다.

리뷰 과정에서는 Ledger를 “record와 artifact를 관리하는 경계”, PromotionPolicy를 “training 가능성을
판단하는 경계”로 구분했습니다. 이에 따라 팀원도 promotion 문제를 persistence 결함과 정책 변경으로
나누어 논의할 수 있고, `Rule → Finding → Decision`이라는 공통 용어로 변경 영향과 테스트 범위를
설명할 수 있습니다. 이는 코드 리뷰에서 조건문 구현보다 정책의 누락·우선순위·안전한 기본값에
집중하게 하는 효과가 있습니다.

## 5. 변경 규모와 범위

| 항목 | 결과 |
| --- | ---: |
| 추가 | 316줄 |
| 삭제 | 38줄 |
| 합계 | 354줄 |
| 파일 | 3개 |
| 허용 목록 외 변경 | 없음 |

변경 파일은 `brainwash/memory/ledger.py`, `brainwash/memory/promotion.py`,
`tests/test_memory_ledger.py`입니다. 300줄을 넘었지만 Policy·Rule Strategy, Decision Value Object,
ledger delegation, 다섯 policy boundary test를 한 coherent unit으로 완결한 결과입니다. 가독성을
해치며 쪼개지 않았습니다.

## 6. 리뷰 결과

리뷰에서는 ledger와 promotion의 책임 경계, conflict/high-risk가 automatic training에 미치는 영향,
기존 `promotion_report(many_min)` 및 Update DB ingestion 보존을 확인했습니다. 세 개의 설계·동작
스레드에서 정책의 이유와 override 범위를 검토했고, test와 전체 suite 검증 결과를 근거로 승인했습니다.

## Black 포맷 검증

각 코드 커밋 직전에 Black 26.5.1을 적용했습니다. 변경된 Python 파일의 comment token과 module/class/function docstring은 제거했으며, SQL·script template·test fixture 같은 실행용 multiline string 값은 보존했습니다. 전체 51개 커밋의 변경 Python blob 57개가 `black --check`를 통과했고, 원본과 재작성본의 실행 AST도 동일합니다.
